# **Event-Driven Video Processing Pipeline**
![Architecture Diagram](https://res.cloudinary.com/db3ogkhvu/image/upload/v1771563142/image_2_rf4fyp.png)

![Architecture Diagram]("./images/Architecture-Diagram.png")

In this project, an event-driven, scalable video processing and streaming platform using AWS.

Video processing requires massive compute power. If we processed videos directly on the web server, the website would freeze for all other users. Instead, we use a  **decoupled architecture**.

----------

**How It Works:**

1.  **The Web Server (EC2 + Flask):**  Serves the UI. When a user uploads an MP4, it uploads the file directly to an Input S3 Bucket and saves a "Queued" status in DynamoDB.
2.  **The Buffer (S3 Events + SQS):**  The moment the video hits S3, S3 automatically fires an event to an SQS Queue. This queue holds the job safely until a worker is ready.
3.  **The Worker (EC2 + FFmpeg):**  A dedicated background server constantly polls the SQS Queue. It downloads the raw video, uses FFmpeg to convert it into a streamable HLS format (`.m3u8`), uploads the chunks to an Output S3 Bucket, and updates DynamoDB to "Ready".
4.  **The Player:**  The web browser reads the "Ready" status and streams the optimized video directly from the S3 Output bucket.
**Goal:**  Create a "Service Role" that grants our EC2 instances permission to talk to S3, SQS, and DynamoDB safely without needing to hardcode secret access keys in our code.

## TASK 1 : SECURITY ( IAM )

**Step 1: Create the Role**  
Navigate to  **IAM**  ->  **Roles**  ->  **Create role**.

**Step 2: Select Trusted Entity**

-   **Trusted entity type:**  Select  **AWS service**.
-   **Service or use case:**  Select  **EC2**  and click Next.

**Step 3: Add Permissions**  
Search for and select these 3 specific policies using the checkboxes:

1.  `AmazonS3FullAccess`
2.  `AmazonSQSFullAccess`
3.  `AmazonDynamoDBFullAccess`

-   Click  **Next**.

**Step 4: Name and Create**

-   **Role name:**  `iam_role_video_lab`
-   Click  **Create role**.

## TASK 2: STORAGE ( S3 )
**Goal:**  Set up two distinct S3 buckets: one to catch the raw uploaded video files, and a second to serve the final optimized stream.

**Step 1: Create the Input Bucket**  
Navigate to  **S3**  ->  **Create bucket**.  
_(Note: S3 bucket names must be globally unique. Add some random numbers/letters to the end of your names!)_

-   **Bucket name:**  `lab-video-input-[your-unique-id]`
-   Leave all other settings as default and click  **Create bucket**.

**Step 2: Create the Output Bucket**  
Click  **Create bucket**  again.

-   **Bucket name:**  `lab-video-output-[your-unique-id]`
-   Leave all other settings as default and click  **Create bucket**.

## TASK 3: DATABASE (DYNAMODB)  

**Goal:**  Create a high-speed NoSQL database table. Our frontend application will read this table to display the live status of the video conversion process to the user.


**Step 1: Create DynamoDB Table**  
Navigate to  **DynamoDB**  ->  **Tables**  ->  **Create table**.

-   **Table name:**  `VideoCatalog`
-   **Partition key:**  `video_id`  (Type:  **String**)

**Step 2: Save and Verify**

-   Leave all other settings as default and click  **Create table**.
-   Wait a few moments until the table's status changes from "Creating" to "Active".  

## TASK 4: CREATE BUFFER (SQS)

  

**Goal:**  Create a message queue and configure your Input S3 Bucket to automatically push a job to this queue every time a new video is uploaded.


**Step 1: Create SQS Queue**  
Navigate to  **SQS**  ->  **Create queue**.

-   **Type:**  Standard
-   **Name:**  `VideoQueue`

**Step 2: Configure Access Policy**  
Scroll down to  **Access policy**  and choose  **Advanced**.

-   **Important Trick:**  Before you delete the default JSON code, look inside it and  **copy the "Resource" value**  (your exact Queue ARN).
-   Now, replace the entire default JSON block with the policy below.
-   Replace the  `Resource`  string with the Queue ARN you just copied, and replace the  `SourceArn`  string with your actual Input Bucket name.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "s3.amazonaws.com" },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:123456789012:VideoQueue",
    "Condition": {
      "ArnLike": { "aws:SourceArn": "arn:aws:s3:::lab-video-input-[your-unique-id]" }
    }
  }]
}
```

-   Click  **Create queue**. Copy your  **Queue URL**  from the details page, you will need it for your Python code!

**Step 3: Link S3 to SQS**

1.  Go to  **S3**  -> Click your  **Input Bucket**  ->  **Properties**  tab.
2.  Scroll down to  **Event notifications**  and click  **Create event notification**.
3.  Name:  `VideoUploadEvent`
4.  **Event types:**  Check the box for  **All object create events**.
5.  **Destination:**  Select  **SQS Queue**  -> Choose  `VideoQueue`.
6.  Click  **Save changes**.

## TASK 5: COMPUTE INFRASTRUCTURE ( EC2 )

  

**Goal:**  Launch the two servers that will power our application: the frontend Web-Server, and the backend Worker-Server.


**Step 1: Launch the Worker-Server**  
Navigate to  **EC2**  ->  **Launch instances**.

-   **Name:**  `Worker-Server`
-   **AMI:**  Amazon Linux 2023
-   **Instance type:**  `t3.micro`
-   **Key pair:**  Select  **Proceed without a key pair**  (We will use EC2 Instance Connect to access the terminal securely from the browser. If you prefer to use the terminal, proceed with creating a key pair and download it).
-   **Network settings:**  Click  **Edit**. Create a security group with the following Inbound Security Group Rules:
    -   **Type:**  SSH |  **Source type:**  Anywhere
    -   **Type:**  HTTP |  **Source type:**  Anywhere
-   **Advanced Details:**  Scroll down to  **IAM instance profile**  and select the  `iam_role_video_lab`  role you created earlier.
-   Click  **Launch instance**.

**Step 2: Launch the Web-Server**  
Click  **Launch instances**  again to create your second server.

-   **Name:**  `Web-Server`
-   **AMI:**  Amazon Linux 2023
-   **Instance type:**  `t3.micro`
-   **Key pair:**  Select  **Proceed without a key pair**  (or select the one you created).
-   **Network settings:**  Click  **Edit**. Create a security group with the following Inbound Security Group Rules:
    -   **Type:**  SSH |  **Source type:**  Anywhere
    -   **Type:**  HTTP |  **Source type:**  Anywhere
-   **Advanced Details:**  Scroll down to  **IAM instance profile**  and select the  `iam_role_video_lab`  IAM instance profile.
-   Click  **Launch instance**.

## TASK 6: DEPLOY THE WORKER

**Goal:**  Install FFmpeg (the industry standard video processor) and deploy the Python script that listens to SQS and converts our videos.


**Step 1: Connect to the Worker-Server**  
Go to your EC2 instances list, select  **Worker-Server**, and click  **Connect**  at the top. Choose  **EC2 Instance Connect**  and click Connect to open a browser terminal.

**Step 2: Install Dependencies**  
Run the following commands block-by-block to install FFmpeg and Python tools:

```bash
cd ~
wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz
tar -xf ffmpeg-release-amd64-static.tar.xz
sudo mv ffmpeg-*-amd64-static/ffmpeg /usr/bin/ffmpeg
sudo mv ffmpeg-*-amd64-static/ffprobe /usr/bin/ffprobe
rm -rf ffmpeg-release-amd64-static.tar.xz ffmpeg-*-amd64-static

sudo dnf install -y python3.11-pip
python3.11 -m pip install boto3
```

**Step 3: Create the Script**  
Open a new file using the nano text editor:

```bash
nano worker.py
```

Paste the Python code below.

-   **CRITICAL:**  You must update the  `INPUT_BUCKET`,  `OUTPUT_BUCKET`, and  `QUEUE_URL`  variables at the top of the script with your exact values!

```python
import boto3, json, os, subprocess, re
from urllib.parse import unquote_plus

INPUT_BUCKET = "lab-video-input-[unique-id]"
OUTPUT_BUCKET = "lab-video-output-[unique-id]"
QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/[id]/VideoQueue"
REGION = "us-east-1"

sqs = boto3.client('sqs', region_name=REGION)
s3 = boto3.client('s3', region_name=REGION)
table = boto3.resource('dynamodb', region_name=REGION).Table('VideoCatalog')

def get_duration(file):
    cmd = ['ffprobe', '-v', 'error', '-show_entries', 'format=duration', '-of', 'default=noprint_wrappers=1:nokey=1', file]
    return float(subprocess.check_output(cmd))

def process():
    print("Worker started. Listening...")
    while True:
        resp = sqs.receive_message(QueueUrl=QUEUE_URL, MaxNumberOfMessages=1, WaitTimeSeconds=20)
        if 'Messages' in resp:
            msg = resp['Messages'][0]
            body = json.loads(msg['Body'])
            local_in = None

            try:
                if 'Event' in body and body['Event'] == 's3:TestEvent':
                    print("Received S3 Test Event. Clearing...")
                    sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg['ReceiptHandle'])
                    continue

                key = unquote_plus(body['Records'][0]['s3']['object']['key'])
                vid = key.split('.')[0].replace(' ', '_')
                local_in, out_dir = f"in_{vid}.mp4", f"out_{vid}"

                print(f"\n--- Processing: {key} ---")
                s3.download_file(INPUT_BUCKET, key, local_in)
                os.makedirs(out_dir, exist_ok=True)

                dur = get_duration(local_in)
                cmd = ['ffmpeg', '-i', local_in, '-profile:v', 'baseline', '-level', '3.0', '-f', 'hls', f'{out_dir}/playlist.m3u8']

                proc = subprocess.Popen(cmd, stderr=subprocess.PIPE, universal_newlines=True)
                for line in proc.stderr:
                    if "time=" in line:
                        m = re.search(r"time=(\d{2}):(\d{2}):(\d{2}\.\d{2})", line)
                        if m:
                            h, mins, s = map(float, m.groups())
                            pct = min(int(((h*3600 + mins*60 + s) / dur) * 90), 90)
                            print(f"   > Progress: {pct}%", end="\r")
                            table.update_item(Key={'video_id':vid}, UpdateExpression="SET #s=:s, progress=:p", ExpressionAttributeNames={'#s':'status'}, ExpressionAttributeValues={':s':'Processing', ':p':pct})
                proc.wait()
                for f in os.listdir(out_dir): s3.upload_file(f"{out_dir}/{f}", OUTPUT_BUCKET, f"{vid}/{f}")
                table.put_item(Item={'video_id':vid, 'status':'Ready', 'progress':100})
                sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg['ReceiptHandle'])
                print(f"\n--- Finished: {vid} ---")

            except Exception as e: 
                print(f"Error: {e}")
            finally:
                if local_in and os.path.exists(local_in): 
                    os.remove(local_in)

process()
```

Press  `Ctrl + X`  to exit, press  `Y`  to save, and press  `Enter`.

**Step 4: Start the Worker**  
Run the script. It will print "Worker started. Listening…" and hang there waiting for SQS messages. Leave this tab open!

```bash
python3.11 worker.py
```

## TASK 7: DEPLOY FRONTEND SERVER

**Goal:**  Set up the Flask web server to serve the UI, handle direct S3 file uploads, and stream the finalized HLS video chunks.

----------

**Step 1: Connect to the Web-Server**  
Go back to the AWS Console, select your  **Web-Server**  EC2 instance, and connect via  **EC2 Instance Connect**.

**Step 2: Install Dependencies**

```bash
sudo dnf install -y python3.11-pip
sudo python3.11 -m pip install flask boto3
```

**Step 3: Create the Server Script**

```bash
nano server.py
```

Paste the code below.  **CRITICAL:**  Update the  `INPUT_BUCKET`  and  `OUTPUT_BUCKET`  variables at the top of the script!

```python
from flask import Flask, render_template_string, jsonify, Response, stream_with_context, request, redirect
import boto3, os

app = Flask(__name__)

INPUT_BUCKET = "lab-video-input-[unique-id]"
OUTPUT_BUCKET = "lab-video-output-[unique-id]"
REGION = "us-east-1"

s3 = boto3.client('s3', region_name=REGION)
dynamodb = boto3.resource('dynamodb', region_name=REGION)
table = dynamodb.Table('VideoCatalog')

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>KodeTube</title>
    <link href="https://vjs.zencdn.net/7.20.3/video-js.css" rel="stylesheet" />
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Inter', sans-serif; background: linear-gradient(135deg, #1e1b4b 0%, #0f172a 50%, #1e293b 100%); color: #f1f5f9; min-height: 100vh; }
        .navbar { display: flex; justify-content: space-between; align-items: center; padding: 20px 60px; background: rgba(15, 23, 42, 0.8); backdrop-filter: blur(20px); border-bottom: 1px solid rgba(99, 102, 241, 0.2); position: sticky; top: 0; z-index: 100; }
        .logo { font-size: 28px; font-weight: 700; background: linear-gradient(135deg, #6366f1, #ec4899); -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
        .btn { background: linear-gradient(135deg, #6366f1, #ec4899); border: none; color: white; padding: 12px 28px; border-radius: 12px; cursor: pointer; font-weight: 600; transition: all 0.3s; box-shadow: 0 4px 15px rgba(99, 102, 241, 0.4); }
        .btn:hover { transform: translateY(-2px); box-shadow: 0 6px 25px rgba(99, 102, 241, 0.6); }
        .container { max-width: 1400px; margin: 50px auto; padding: 0 40px; }
        .video-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(320px, 1fr)); gap: 30px; }
        .card { background: rgba(30, 41, 59, 0.5); backdrop-filter: blur(10px); border-radius: 20px; border: 1px solid rgba(99, 102, 241, 0.2); cursor: pointer; transition: all 0.4s; position: relative; overflow: hidden; box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3); }
        .card:hover { border-color: #6366f1; transform: translateY(-8px); box-shadow: 0 20px 60px rgba(99, 102, 241, 0.4); }
        .thumb { height: 200px; background: linear-gradient(135deg, rgba(99, 102, 241, 0.2), rgba(236, 72, 153, 0.2)); display: flex; align-items: center; justify-content: center; font-size: 64px; }
        .info { padding: 20px; }
        .info strong { display: block; font-size: 16px; font-weight: 700; margin-bottom: 8px; }
        .status-badge { display: inline-block; padding: 4px 12px; border-radius: 20px; font-size: 12px; font-weight: 600; }
        .status-ready { background: rgba(16, 185, 129, 0.2); color: #10b981; border: 1px solid #10b981; }
        .status-processing { background: rgba(245, 158, 11, 0.2); color: #f59e0b; border: 1px solid #f59e0b; }
        .delete-btn { position: absolute; top: 12px; right: 12px; background: rgba(239, 68, 68, 0.9); color: white; border: none; border-radius: 50%; width: 36px; height: 36px; cursor: pointer; z-index: 10; opacity: 0; transition: all 0.3s; font-size: 20px; }
        .card:hover .delete-btn { opacity: 1; }
        .delete-btn:hover { transform: scale(1.1) rotate(90deg); }
        .modal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.85); backdrop-filter: blur(10px); z-index: 1000; justify-content: center; align-items: center; }
        .modal-content { background: rgba(30, 41, 59, 0.95); backdrop-filter: blur(20px); padding: 40px; border-radius: 24px; width: 90%; max-width: 900px; border: 1px solid rgba(99, 102, 241, 0.3); text-align: center; }
        .modal-content h2 { font-size: 28px; font-weight: 700; margin-bottom: 30px; background: linear-gradient(135deg, #6366f1, #ec4899); -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
        .video-js { width: 100% !important; height: auto !important; aspect-ratio: 16/9; border-radius: 16px; }
        #uploadingState { display: none; }
        .spinner { border: 5px solid rgba(99, 102, 241, 0.1); border-top: 5px solid #6366f1; border-radius: 50%; width: 60px; height: 60px; animation: spin 0.8s linear infinite; margin: 30px auto; }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
        input[type="file"] { margin: 20px 0; padding: 15px; background: rgba(99, 102, 241, 0.1); border: 2px dashed rgba(99, 102, 241, 0.4); border-radius: 12px; color: #f1f5f9; width: 100%; cursor: pointer; transition: all 0.3s; }
        input[type="file"]:hover { border-color: #6366f1; background: rgba(99, 102, 241, 0.2); }
    </style>
</head>
<body>
    <div class="navbar">
        <div class="logo">KODETUBE</div>
        <button class="btn" onclick="openUpload()">+ UPLOAD VIDEO</button>
    </div>
    <div class="container"><div class="video-grid" id="grid"></div></div>

    <div id="uploadModal" class="modal" onclick="document.getElementById('uploadModal').style.display='none';">
        <div class="modal-content" style="max-width: 400px;" onclick="event.stopPropagation()">
            <button onclick="document.getElementById('uploadModal').style.display='none';" style="position: absolute; top: 15px; right: 15px; background: rgba(239, 68, 68, 0.9); color: white; border: none; border-radius: 50%; width: 32px; height: 32px; cursor: pointer; font-size: 18px; transition: all 0.3s;">&times;</button>
            <div id="formState">
                <h2>New Deployment</h2>
                <form id="uploadForm">
                    <input type="file" id="fileInput" name="file" accept=".mp4" style="margin:20px 0; color:#ccc;" required><br>
                    <button type="submit" class="btn">START UPLOAD</button>
                </form>
            </div>
            <div id="uploadingState">
                <div class="spinner"></div>
                <p>Transferring to S3...</p>
            </div>
        </div>
    </div>

    <div id="playerModal" class="modal" onclick="closePlayer()">
        <div class="modal-content" onclick="event.stopPropagation()">
            <video id="my-video" class="video-js vjs-big-play-centered" controls preload="auto"></video>
            <h3 id="videoTitle" style="margin-top:20px; color: var(--kk-cyan);"></h3>
        </div>
    </div>

    <script src="https://vjs.zencdn.net/7.20.3/video.min.js"></script>
    <script>
        function openUpload() { document.getElementById('uploadModal').style.display = 'flex'; }

        document.getElementById('uploadForm').onsubmit = async (e) => {
            e.preventDefault();
            const file = document.getElementById('fileInput').files[0];
            if(!file) return;

            document.getElementById('formState').style.display = 'none';
            document.getElementById('uploadingState').style.display = 'block';

            const formData = new FormData();
            formData.append('file', file);

            try {
                const response = await fetch('/upload', { method: 'POST', body: formData });
                if (response.ok) {
                    document.getElementById('uploadModal').style.display = 'none';
                    document.getElementById('uploadingState').style.display = 'none';
                    document.getElementById('formState').style.display = 'block';
                    document.getElementById('fileInput').value = '';
                    loadVideos();
                } else { alert("Upload failed on server."); }
            } catch (err) { alert("Error connecting to server."); }
        };

        function loadVideos() {
            fetch('/api/videos?t=' + Date.now()).then(r => r.json()).then(videos => {
                const grid = document.getElementById('grid');
                if (videos.length === 0) {
                    grid.innerHTML = `
                        <div style="grid-column: 1/-1; text-align: center; padding: 80px 20px;">
                            <div style="font-size: 120px; margin-bottom: 20px; opacity: 0.3;">🎬</div>
                            <h2 style="font-size: 32px; font-weight: 700; margin-bottom: 15px; background: linear-gradient(135deg, #6366f1, #ec4899); -webkit-background-clip: text; -webkit-text-fill-color: transparent;">No Videos Yet</h2>
                            <p style="color: #94a3b8; font-size: 18px; margin-bottom: 30px;">Upload your first video to get started</p>
                            <button class="btn" onclick="openUpload()" style="font-size: 16px; padding: 14px 32px;">+ Upload Video</button>
                        </div>`;
                    return;
                }
                grid.innerHTML = '';
                videos.forEach(v => {
                    const isReady = v.status === 'Ready';
                    const statusClass = isReady ? 'status-ready' : 'status-processing';
                    const icon = isReady ? '▶' : '⚙';
                    grid.innerHTML += `
                        <div class="card" onclick="playVideo('${v.video_id}', '${v.status}')">
                            <button class="delete-btn" onclick="event.stopPropagation(); deleteVideo('${v.video_id}')">&times;</button>
                            <div class="thumb"><span class="thumb-icon">${icon}</span></div>
                            <div class="info">
                                <strong>${v.video_id}</strong>
                                <span class="status-badge ${statusClass}">${v.status} ${v.progress || 0}%</span>
                            </div>
                        </div>`;
                });
            });
        }

        function playVideo(id, status) {
            if (status !== 'Ready') return;
            const player = videojs('my-video');
            player.src({ type: 'application/x-mpegURL', src: '/stream/' + encodeURIComponent(id) + '/playlist.m3u8' });
            document.getElementById('videoTitle').innerText = id;
            document.getElementById('playerModal').style.display = 'flex';
            player.play();
        }

        function closePlayer() { document.getElementById('playerModal').style.display = 'none'; videojs('my-video').pause(); }

        async function deleteVideo(id) {
            if (!confirm("Delete this video?")) return;
            await fetch('/api/delete/' + encodeURIComponent(id), { method: 'POST' });
            loadVideos();
        }

        loadVideos();
        setInterval(loadVideos, 3000); 
    </script>
</body>
</html>
"""

@app.route('/')
def home(): return render_template_string(HTML_TEMPLATE)

@app.route('/api/videos')
def list_videos():
    items = table.scan().get('Items', [])
    items.sort(key=lambda x: x['video_id'])
    return jsonify(items)

@app.route('/upload', methods=['POST'])
def upload():
    file = request.files.get('file')
    if file:
        s3.upload_fileobj(file, INPUT_BUCKET, file.filename)
        video_id = file.filename.split('.')[0].replace(' ', '_').replace('+', '_')
        table.put_item(Item={'video_id': video_id, 'status': 'Queued', 'progress': 0})
        return jsonify({"status": "success"}), 200
    return jsonify({"status": "error"}), 400

@app.route('/stream/<video_id>/<path:filename>')
def stream(video_id, filename):
    key = f"{video_id}/{filename}"
    content_type = 'application/x-mpegURL' if filename.endswith('.m3u8') else 'video/MP2T'
    try:
        s3_obj = s3.get_object(Bucket=OUTPUT_BUCKET, Key=key)
        return Response(stream_with_context(s3_obj['Body'].iter_chunks(chunk_size=4096)), mimetype=content_type)
    except: return Response("Not Found", status=404)

@app.route('/api/delete/<video_id>', methods=['POST'])
def delete_video(video_id):
    table.delete_item(Key={'video_id': video_id})
    objs = s3.list_objects_v2(Bucket=OUTPUT_BUCKET, Prefix=f"{video_id}/")
    if 'Contents' in objs:
        s3.delete_objects(Bucket=OUTPUT_BUCKET, Delete={'Objects': [{'Key': o['Key']} for o in objs['Contents']]})
    return jsonify({"status": "deleted"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

Press  `Ctrl + X`  to exit, press  `Y`  to save, and press  `Enter`.

**Step 4: Start the Server**  
Run the Flask server. We use sudo because the script needs permission to open port 80 for HTTP traffic.

```bash
sudo python3.11 server.py
```

## TASK 8: APPLICATION VALIDATION

**Time to test the pipeline!**

**Step 1: Open the App**

1.  Go back to your EC2 console and locate your  **Web-Server**  instance.
2.  Copy its  **Public IPv4 address**.
3.  Paste the IP address into a new tab in your web browser. You should see the interface!

**Step 2: Upload a Video**

1.  Click the  **+ UPLOAD VIDEO**  button.
2.  Select a short  `.mp4`  video from your computer (using a small file is recommended so you don't have to wait too long!).
3.  The Web-Server will upload the file to your Input S3 bucket and log it as "Queued".

**Step 3: Watch the Worker**  
If you switch back to the browser tab where your  **Worker-Server**  terminal is running, you will see it instantly pick up the job and start printing the conversion progress percentage!

**Step 4: Play**  
Once the worker hits 100%, the UI will update the status to "Ready". Click the video card to stream your fully processed, chunked HLS video directly from your Output S3 bucket.

------------
---------------


  
## MISSION ACCOMPLISHED


By separating the web interface from the heavy video processing logic, you ensured that the application remains highly responsive, no matter how large the uploaded files are.

**What I Have Mastered:**

-   **Asynchronous Processing:**  You utilized SQS to safely buffer incoming jobs.
-   **Event-Driven Architecture:**  You configured S3 to trigger workflows automatically using Event Notifications.
-   **State Management:**  You used DynamoDB to track the live progress of background tasks and reflect them on the frontend.
-   **Compute Separation:**  You deployed distinct EC2 resources with IAM roles, allowing frontend and backend logic to scale independently without bottlenecking each other.