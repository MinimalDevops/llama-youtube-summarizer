# YouTube Video Summarizer

This Python project provides a Streamlit-based application that processes YouTube videos to extract audio, transcribe it into text using OpenAI Whisper, and summarize the content into a concise format. It supports generating PDF files for the transcription and summary.

## Features

1. **Convert YouTube Shorts URLs:** Converts YouTube Shorts URLs to standard video URLs.
2. **Audio Download:** Downloads audio from YouTube videos using `yt-dlp`.
3. **Audio Transcription:** Transcribes audio to text using OpenAI Whisper.
4. **PDF Creation:** Converts transcribed text and summaries into PDF files.
5. **Summarization:** Summarizes transcribed content using a local summarization API.
6. **Streamlit Interface:** User-friendly interface for providing YouTube URLs and downloading summaries.

## Requirements

### Python Libraries
- `os`
- `subprocess`
- `whisper`
- `ssl`
- `urllib`
- `imageio_ffmpeg`
- `re`
- `time`
- `yt_dlp`
- `sys`
- `streamlit`
- `fpdf`
- `requests`
- `asyncio`

### External Tools
- **FFmpeg**: Required for audio processing.
- **OpenAI Whisper**: For transcription.

## Code Explanation (Line-by-Line)

### Base URL for Summarization
```python
base_url = "http://127.0.0.1:8001"
```
Defines the base URL for the local summarization API, which handles ingestion and summarization requests for PDF files.

### Convert YouTube Shorts URL
```python
def convert_shorts_url(url):
    if "youtube.com/shorts/" in url:
        return re.sub(r"/shorts/", "/watch?v=", url)
    return url
```
- Checks if the URL contains the string `/shorts/`.
- If true, replaces `/shorts/` with `/watch?v=` to standardize the URL for processing.
- Returns the modified or original URL.

### Download Audio
```python
def download_audio(youtube_url, output_path="audio.mp3"):
    try:
        print("Attempting to download audio from YouTube using yt-dlp...")
        if os.path.exists(output_path):
            os.remove(output_path)
        ydl_opts = {
            'format': 'bestaudio/best',
            'outtmpl': output_path.replace('.mp3', '') + '.%(ext)s',
            'postprocessors': [{
                'key': 'FFmpegExtractAudio',
                'preferredcodec': 'mp3',
                'preferredquality': '192',
            }],
            'nocheckcertificate': True,
            'ffmpeg_location': ffmpeg.get_ffmpeg_exe(),
        }
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([youtube_url])
        print(f"Audio successfully downloaded as {output_path}")
        return True
    except Exception as e:
        print(f"An error occurred while downloading audio: {e}")
        return False
```
- Uses `yt-dlp` to extract audio from the provided YouTube URL.
- Configures FFmpeg to convert audio to MP3 format with 192 kbps quality.
- Removes any existing audio file with the same name before downloading.
- Returns `True` if successful, otherwise logs the error.

### Get Video Title
```python
def get_video_title(youtube_url):
    try:
        ydl_opts = {'quiet': True, 'skip_download': True}
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(youtube_url, download=False)
            title = info.get("title", "summary")
            print(f"Fetched video title: {title}")
            return title
    except Exception as e:
        print(f"An error occurred while retrieving video title: {e}")
        return "summary"
```
- Uses `yt-dlp` to fetch metadata for the provided YouTube URL.
- Extracts the title of the video, which is used to name the generated PDF files.
- Returns the title or a default string `"summary"` if an error occurs.

### Transcribe Audio
```python
def transcribe_audio(audio_path):
    try:
        print("Attempting to transcribe audio using Whisper...")
        ffmpeg_path = ffmpeg.get_ffmpeg_exe()
        os.environ["PATH"] += os.pathsep + os.path.dirname(ffmpeg_path)
        os.environ["FFMPEG_BINARY"] = ffmpeg_path
        whisper.audio.FFMPEG = ffmpeg_path
        wav_path = "youtube_audio.wav"
        if os.path.exists(wav_path):
            os.remove(wav_path)
        subprocess.run([ffmpeg_path, "-i", audio_path, wav_path], check=True)
        model = whisper.load_model("base")
        result = model.transcribe(wav_path, fp16=False)
        print("Transcription:")
        print(result["text"])
        return result["text"]
    except Exception as e:
        print(f"An error occurred during transcription: {e}")
        return None
```
- Uses FFmpeg to convert the audio file to WAV format, required by Whisper.
- Loads a base Whisper model to transcribe the WAV file into text.
- Returns the transcription or logs an error if the process fails.

### Create PDF
```python
def create_pdf(transcription, pdf_path, youtube_url):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    pdf.multi_cell(0, 10, f"YouTube URL: {youtube_url}\n\n{transcription}")
    if not os.path.exists(os.path.dirname(pdf_path)):
        os.makedirs(os.path.dirname(pdf_path))
    pdf.output(pdf_path)
    print(f"PDF created at {pdf_path}")
```
- Creates a PDF with the provided transcription and the YouTube URL.
- Uses `FPDF` for layout and text rendering.
- Saves the PDF in the specified path, creating directories if needed.

### Summarize PDF
```python
def summarize_pdf(pdf_path):
    print(f"Starting summarization for {pdf_path}")
    ingest_url = f"{base_url}/v1/ingest/file"
    with open(pdf_path, 'rb') as file:
        ingest_response = requests.post(ingest_url, files={'file': file})

    if ingest_response.status_code == 200:
        print(f"PDF {pdf_path} ingested successfully.")
    else:
        print(f"Failed to ingest PDF: {ingest_response.status_code}")
        return None

    summarize_url = f"{base_url}/v1/summarize"
    headers = {"accept": "application/json", "Content-Type": "application/json"}
    payload = {
        "text": "string",
        "use_context": True,
        "prompt": ("Provide a comprehensive summary of the provided context information. The summary should cover all key points."),
        "stream": False
    }

    try:
        response = requests.post(summarize_url, headers=headers, json=payload)
        response.raise_for_status()
        summary = response.json().get("summary", "No summary found")
        print(f"Summary: {summary}")
        return summary
    except requests.exceptions.RequestException as e:
        print(f"Error during summarization: {e}")
        return None
```
- Ingests the PDF into the summarization API.
- Sends a request with the summarization prompt and retrieves the summary as JSON.
- Logs and returns the summary, or logs an error if the process fails.

### Delete Ingested Document
```python
def delete_ingested_document(pdf_name):
    list_url = f"{base_url}/v1/ingest/list"
    list_response = requests.get(list_url)

    if list_response.status_code == 200:
        list_data = list_response.json()
        doc_ids_to_delete = [
            document['doc_id']
            for document in list_data.get('data', [])
            if document['doc_metadata']['file_name'] == pdf_name
        ]

        for doc_id in doc_ids_to_delete:
            delete_url = f"{base_url}/v1/ingest/{doc_id}"
            delete_response = requests.delete(delete_url)

            if delete_response.status_code == 200:
                print(f"Document with doc_id {doc_id} successfully deleted.")
            else:
                print(f"Failed to delete document with doc_id {doc_id}: {delete_response.status_code}")
    else:
        print(f"Failed to fetch document list: {list_response.status_code}")
```
- Lists all ingested documents via the API.
- Matches and deletes the specified PDF using its `doc_id`.
- Logs success or failure messages for each deletion request.

### Streamlit App
```python
def main():
    st.title("YouTube Video Summarizer 🎥")
    st.markdown('<style>h1{color: orange; text-align: center;}</style>', unsafe_allow_html=True)

    with st.expander("About the App"):
        st.write("This app allows you to summarize the audio of a YouTube video.")
        st.write("Enter a YouTube URL in the input box below and click 'Submit' to start.")

    youtube_url = st.text_input("Enter YouTube URL")

    if st.button("Submit") and youtube_url:
        start_time = time.time()
        youtube_url = convert_shorts_url(youtube_url)
        video_title = get_video_title(youtube_url)
        print(f"Debug: Video title is '{video_title}'")

        audio_output_path = "youtube_audio.mp3"
        if download_audio(youtube_url, audio_output_path):
            if os.path.exists(audio_output_path):
                output = transcribe_audio(audio_output_path)
                if output:
                    pdf_path = f"pdf/{video_title}.pdf"
                    create_pdf(output, pdf_path, youtube_url)

                    summary = summarize_pdf(pdf_path)

                    col1, col2 = st.columns([1, 1])

                    with col1:
                        st.video(youtube_url)

                    with col2:
                        st.header("Summarization of YouTube Video")
                        if summary:
                            st.write(summary)
                            summary_pdf_path = f"pdf/{video_title}_summary.pdf"
                            create_pdf(summary, summary_pdf_path, youtube_url)

                            with open(summary_pdf_path, "rb") as pdf_file:
                                st.download_button(
                                    label="Download Summary as PDF",
                                    data=pdf_file,
                                    file_name=f"{video_title}_summary.pdf",
                                    mime="application/pdf",
                                    on_click=None
                                )

                    delete_ingested_document(os.path.basename(pdf_path))

        end_time = time.time()
        elapsed_time = end_time - start_time
        st.write(f"Time taken: {elapsed_time:.2f} seconds")
```
- Implements the Streamlit-based user interface.
- Allows users to input a YouTube URL and trigger the summarization pipeline.
- Displays the video and its summary side by side.
- Provides a download button for the summary PDF.

## How to Run

1. Clone the repository.
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Start the Streamlit app:
   ```bash
   streamlit run app.py
   ```
4. Access the app in your browser at `http://localhost:8501`.

## Folder Structure
- **audio.mp3**: Temporary audio file.
- **youtube_audio.wav**: Temporary WAV file for transcription.
- **pdf/**: Stores generated PDFs.

## Troubleshooting

- Ensure FFmpeg is installed and accessible.
- Check if the local summarization API is running.
- Verify internet connection for YouTube video processing.

## Future Enhancements

- Add support for multiple languages in transcription and summarization.
- Integrate cloud-based summarization APIs for broader use cases.
- Enhance UI with more user controls and feedback.

## License
MIT License
