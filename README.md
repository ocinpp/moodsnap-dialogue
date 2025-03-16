# MoodSnap Movie Quote Generator

MoodSnap is an interactive web application that analyzes your mood from a photo and overlays a matching movie quote. Capture a selfie or upload an image, detect your emotion, and download the result with a stylish quote overlay!

## Features

- **Photo Input:** Take a photo via webcam or upload an image file.
- **Mood Detection:** Uses `@vladmandic/human` to detect emotions (happy, sad, angry, surprised, neutral) from facial expressions.
- **Quote Overlay:** Displays one of 60 curated movie quotes (12 per emotion: 6 English, 6 Cantonese) with original language movie and character names.
- **Styling:** Random, distinct background colors for the quote and movie/character sections.
- **No Face Detection:** Custom modal prompts to retry or assigns a neutral quote if no face is detected.
- **Download:** Save the photo with quote as a PNG.

## Tech Stack

- **Framework:** Vue.js 3 with Vite
- **AI Library:** `@vladmandic/human` (powered by TensorFlow.js)
- **Image Processing:** `html2canvas`
- **Styling:** Tailwind CSS
- **Font:** Montserrat

## Prerequisites

- **Node.js:** Version 16 or higher (tested with 18)
- **npm:** Version 8 or higher

## Installation

1. **Clone the Repository:**

    ```bash
    git clone <your-repository-url>
    cd moodsnap-movie-quote-generator

2. Install Dependencies:

    ``` bash
    npm install
    ```

    Key packages installed:
    - `vue`, `@vitejs/plugin-vue`
    - `@vladmandic/human`, `@tensorflowtfjs`
    - `html2canvas`
    - `tailwindcss`, `postcss`, `autoprefixer`

3. Start Development Server:

    ```bash
    npm run dev
    ```

    Open your browser to <http://localhost:5173>.

4. Build for Production (Optional):

    ```bash
    npm run build
    ```

    Output is generated in the `dist/` directory.

## Project Structure

```
moodsnap-movie-quote-generator/
├── public/                  # Static assets (e.g., favicon.ico)
├── src/
│   ├── components/
│   │   └── NoFaceModal.vue  # Modal for no-face scenarios
│   ├── App.vue              # Main application component
│   └── quotes.js            # Movie quote database
├── index.html               # Entry HTML file
├── package.json             # Project metadata and dependencies
├── vite.config.js           # Vite configuration
├── tailwind.config.js       # Tailwind CSS settings
├── postcss.config.js        # PostCSS configuration
└── README.md                # Project documentation
```

## Usage

1. Launch the App:

    Navigate to <http://localhost:5173> after starting the server.

2. Capture or Upload a Photo:

    - Take a Photo: Click "Take a Photo," then "Capture" to use your webcam.
    - Upload a Photo: Click "Upload Photo" and select an image.

3. Analyze Your Mood:

    - Click "Analyze Mood" to process the photo.
    - If no face is detected:
        - A modal appears with options:
            - "Yes, Try Again" resets to the start.
            - "Proceed Anyway" assigns a random neutral quote.

4. View and Download:

    - See your photo with a quote overlay (distinct colors for quote and movie/character).
    - Mood is shown below (e.g., "Mood Detected: Happy").
    - Click "Download" to save as a PNG or "Try Again" to restart.

## Configuration

- Mood Detection:
  - Uses `@vladmandic/human` with WebGL backend by default.
  - Models are fetched from <https://cdn.jsdelivr.net/npm/@vladmandic/human/models/>.
  - For local models:
    1. Download from Human GitHub.
    2. Place in `public/models/`.
    3. Edit `src/App.vue`, change `humanConfig.modelBasePath` to `"/models/"`.
- Quotes:
  - Modify `src/quotes.js` to customize quotes. Structure:

```javascript
{
  quote: "Your quote here",
  movie: "English Movie Name",
  movieOriginal: "Original Language Name",
  character: "English Character Name",
  characterOriginal: "Original Language Name",
  romanization: "Optional Romanization",
  translation: "Optional Translation"
}
```

## Troubleshooting

- **App Doesn’t Load**: Check console for "Human models loaded" log. If missing, ensure internet for CDN or use local models.
- **Webcam Issues**: Verify camera permissions in your browser.
- **Download Fails**: Confirm `html2canvas` works (check console for "Download URL").
- **Performance**: If slow, switch `humanConfig.backend` to `"wasm"` or `"cpu"` in `src/App.vue`.

## Contributing

Contributions welcome! Suggestions:

- Expand quote database with more languages.
- Add animations to the modal.
- Support multiple faces with aggregated mood.

## License

MIT License. See LICENSE if included.

## Credits

- `@vladmandic/human` for advanced mood detection.
- Vue.js and Vite for a modern frontend.
- Tailwind CSS for responsive design.
Happy mood snapping!
