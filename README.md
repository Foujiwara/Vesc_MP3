# Vesc_MP3 project :
______________________________________________________________________________________
______________________________________________________________________________________
QML code : 
______________________________________________________________________________________
______________________________________________________________________________________


(
import QtQuick 2.12
import QtQuick.Controls 2.0
import QtQuick.Layouts 1.3
import QtQuick.Controls.Material 2.2

import Vedder.vesc.commands 1.0
import Vedder.vesc.utility 1.0
import Vedder.vesc.qminimp3 1.0

Item {
    id: mainItem
    anchors.fill: parent
    anchors.margins: 5
    
    property Commands mCommands: VescIf.commands()
    property var dec: []
    property var dec_ofs: 0
    
    property var lastInput: 0
    property var lastOutput: 0
    property var momentum: 0
    
    // Downsample file by this factor. Saves bandwidth, but loses some quality.
    property var downsample_factor: 2
    
    // Motors will have much higher gain at lowe frequencies, so use a high-pass filter on
    // the samples before sending them.
    property var hpf_cut: 0.2
    
    // https://beammyselfintothefuture.wordpress.com/2015/02/16/simple-c-code-for-resonant-lpf-hpf-filters-and-high-low-shelving-eqs/
    function doResonantHPF (input) {
        lastOutput += momentum - lastInput + input;
        lastInput = input;
        // First number controls resonance; second controls cutoff frequency
        var resonance = hpf_cut
        momentum = momentum * resonance - lastOutput * hpf_cut;
        return lastOutput;
    }
    
    QMiniMp3 {
        id: mp3
    }
    
    Component.onCompleted: {
        dec = mp3.decodeFileMono("/home/benjamin/Musik/test.mp3")
    }
    
    ColumnLayout {
        anchors.fill: parent
           
        Text {
            Layout.fillWidth: true
            Layout.fillHeight: true
            text: "Sample server running. The play_mp3.lisp example should work now."
            color: "white"
            wrapMode: Text.WordWrap
        }
        
        Button {
            Layout.fillWidth: true
            text: "Restart"
            
            onClicked: {
                dec_ofs = 0
                lastInput = 0
                lastOutput = 0
                momentum = 0
                
                console.log(dec.samples.length)
                console.log(dec.fSamp)
                console.log(dec.samples.length / dec.fSamp / 60.0)
            }
        }
    }
    
    Connections {
        target: mCommands
        
        function onCustomAppDataReceived(data) {
            var dv = new DataView(data, 0)
            var ind = 0
            
            // TODO: Use received data
            
            var samples_left = dec.samples.length - dec_ofs
            var len = 500
            if (samples_left < (len * downsample_factor)) {
                len = samples_left
            }
            
            var dataTx = new ArrayBuffer(len + 8)
            var dvTx = new DataView(dataTx)
            var indTx = 0
            dvTx.setUint32(ind, samples_left); ind += 4
            dvTx.setUint32(ind, dec.fSamp / downsample_factor); ind += 4
            
            for (let i = 0;i < len;i++) {
                dvTx.setInt8(ind++, doResonantHPF(dec.samples[(i * downsample_factor) + dec_ofs]) * 127)
            }
            
            dec_ofs += len * downsample_factor
            
            mCommands.sendCustomAppData(dataTx)
        }
    }
}
)



______________________________________________________________________________________
______________________________________________________________________________________
Lisp code : 
______________________________________________________________________________________
______________________________________________________________________________________

(
; This examples requires running Mp3Stream.qml in VESC Tool and that
; the connection to VESC Tool is sufficiently fast (e.g. USB)

(def wait-pid -1)

(defun event-handler ()
    (loopwhile t
        (recv
            ((event-data-rx . (? data)) (send wait-pid data))
            (_ nil)
)))

(event-register-handler (spawn event-handler))
(event-enable 'event-data-rx)

(defun get-samples () {
        (setq wait-pid (self))
        (send-data '(1))
        (recv-to 0.001
            (timeout {
                    (recv-to 0.1
                        (timeout (print "Timeout"))
                        ((? data) data)
                    )
            })
            ((? data) data)
        )
})

(defun test-speed () {
        (var start (systime))
        (get-samples)
        (secs-since start)
})

(def chunk-size 500)

(def buf1 (bufcreate chunk-size))
(def buf2 (bufcreate chunk-size))
(def buf3 (bufcreate chunk-size))

(defun play-samples (buf) {
        (var samp (get-samples))
        (def num-samp (bufget-u32 samp 0))
        (def f-samp (bufget-u32 samp 4))
        (bufcpy buf 0 samp 8 chunk-size)
        (free samp)
        
        (foc-play-samples buf f-samp 0.3)
})

(loopwhile t {
        (play-samples buf1)
        (play-samples buf2)
        (play-samples buf3)
})
)
