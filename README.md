import subprocess, base64, os, tempfile, urllib.parse, urllib.request
from flask import Flask, request, jsonify

app = Flask(__name__)

CHAR_BASE = "Korean male, age 25, black short hair, casual hoodie, bright smile, consistent character"

@app.route("/health")
def health():
    return "ok"

@app.route("/make-video", methods=["POST"])
def make_video():
    data = request.json
    scenes = data["scenes"]
    char_extra = data.get("character", "")

    with tempfile.TemporaryDirectory() as tmp:
        clips = []
        for i, scene in enumerate(scenes):
            # 이미지 생성 (Pollinations.ai — 완전 무료)
            prompt = urllib.parse.quote(
                f"{CHAR_BASE}, {char_extra}, {scene['visual']}, 9:16 vertical shorts"
            )
            img_url = f"https://image.pollinations.ai/prompt/{prompt}?width=1080&height=1920&nologo=true"
            img_path = f"{tmp}/img_{i}.jpg"
            urllib.request.urlretrieve(img_url, img_path)

            # 음성 생성 (edge-tts — Microsoft 무료)
            audio_path = f"{tmp}/aud_{i}.mp3"
            subprocess.run([
                "edge-tts", "--voice", "ko-KR-SunHiNeural",
                "--text", scene["dialogue"],
                "--write-media", audio_path
            ], check=True)

            # 자막 포함 영상 클립 생성 (FFmpeg)
            clip_path = f"{tmp}/clip_{i}.mp4"
            caption = scene.get("caption", "").replace("'", "\\'")
            vf = (
                f"scale=1080:1920:force_original_aspect_ratio=increase,"
                f"crop=1080:1920,"
                f"drawtext=text='{caption}':fontsize=52:fontcolor=white:"
                f"x=(w-text_w)/2:y=h*0.85:box=1:boxcolor=black@0.6:boxborderw=12"
            )
            subprocess.run([
                "ffmpeg", "-y", "-loop", "1", "-i", img_path,
                "-i", audio_path, "-vf", vf,
                "-c:v", "libx264", "-c:a", "aac",
                "-shortest", "-pix_fmt", "yuv420p", clip_path
            ], check=True)
            clips.append(clip_path)

        # 클립 이어붙이기
        list_txt = f"{tmp}/list.txt"
        with open(list_txt, "w") as f:
            for c in clips:
                f.write(f"file '{c}'\n")
        out = f"{tmp}/final.mp4"
        subprocess.run([
            "ffmpeg", "-y", "-f", "concat", "-safe", "0",
            "-i", list_txt, "-c", "copy", out
        ], check=True)  

        video_b64 = base64.b64encode(open(out, "rb").read()).decode()
        return jsonify({"video_base64": video_b64, "status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
