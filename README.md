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
