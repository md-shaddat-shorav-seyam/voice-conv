```
# voice-conv
from flask import Flask, request, send_file
from flask_cors import CORS
import uuid
import os
import librosa
import soundfile as sf

app = Flask(__name__)
CORS(app)

# Function to shift pitch and change speed
def process_audio(audio_path, pitch_steps, rate):
    # Load audio file
    y, sr = librosa.load(audio_path, sr=None)

    # Step 1: Pitch shift
    if pitch_steps != 0:
        y = librosa.effects.pitch_shift(y=y, sr=sr, n_steps=pitch_steps)

    # Step 2: Change tempo
    if rate != 1.0:
        y = librosa.effects.time_stretch(y, rate=rate)

    # Save processed audio
    output_path = f"converted_{uuid.uuid4().hex}.wav"
    sf.write(output_path, y, sr)
    return output_path

@app.route('/convert-voice', methods=['POST'])
def convert_voice():
    if 'file' not in request.files:
        return {'error': 'No file uploaded'}, 400

    # Get pitch and rate from request (with defaults)
    try:
        pitch_steps = float(request.form.get('pitch', 4))
        rate = float(request.form.get('rate', 1.05))
    except ValueError:
        return {'error': 'Invalid pitch or rate value'}, 400

    file = request.files['file']
    input_ext = file.filename.split('.')[-1]
    input_filename = f"input_{uuid.uuid4()}.{input_ext}"
    file.save(input_filename)

    try:
        converted_path = process_audio(input_filename, pitch_steps, rate)
        return send_file(converted_path, mimetype='audio/wav', as_attachment=True)

    finally:
        if os.path.exists(input_filename):
            os.remove(input_filename)

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```
### here html

```
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Voice Converter</title>
</head>

<body>
    <h2>Voice Converter</h2>

    <label>Select Audio File:</label>
    <input type="file" id="audioFile" accept="audio/*"><br><br>

    <label>Pitch (semitones):</label>
    <input type="number" id="pitch" step="0.1"><br><br>

    <label>Speed Rate:</label>
    <input type="number" id="rate" step="0.01"><br><br>

    <button onclick="uploadAudio()">Convert Voice</button>

    <h3>Preview</h3>
    <audio id="audioPlayer" controls></audio>

    <script>
        async function uploadAudio() {
            const fileInput = document.getElementById("audioFile");
            const pitchInput = document.getElementById("pitch").value;
            const rateInput = document.getElementById("rate").value;

            if (!fileInput.files.length) {
                alert("Please select an audio file first!");
                return;
            }

            const formData = new FormData();
            formData.append("file", fileInput.files[0]);
            formData.append("pitch", pitchInput);
            formData.append("rate", rateInput);

            try {
                const response = await fetch("http://localhost:5000/convert-voice", {
                    method: "POST",
                    body: formData
                });

                if (!response.ok) {
                    throw new Error(`Server error: ${response.status}`);
                }

                const blob = await response.blob();
                if (blob.size === 0) {
                    throw new Error("Received empty audio file");
                }

                const url = URL.createObjectURL(blob);

                // Play audio in browser
                const audioPlayer = document.getElementById("audioPlayer");
                audioPlayer.src = url;
                audioPlayer.play();

            } catch (error) {
                console.error("Error converting voice:", error);
                alert("Error converting voice. Check console for details.");
            }
        }
    </script>
</body>

</html>
```
