# VIXI – AI Chat Assistant

VIXI is an AI-powered assistant web app that lets you interact via text or voice calls, with support for multiple languages. Built with SvelteKit, Vite, and Tailwind CSS, VIXI provides a seamless and modern conversational experience.

## Features

- **Text Chat**: Chat with VIXI using a simple and intuitive interface.
- **Voice Calls**: Start a call and converse with VIXI using your microphone and speakers.
- **Multilingual Support**: Communicate in various languages—VIXI will understand and respond accordingly.
- **Real-Time Transcription**: Voice input is transcribed and processed instantly.
- **Responsive UI**: Clean, mobile-friendly design powered by Tailwind CSS.

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) (v18+ recommended)
- [pnpm](https://pnpm.io/) or [npm](https://www.npmjs.com/)

### Installation

1. Clone the repository:
   ```sh
   git clone https://github.com/yourusername/vixi.git
   cd vixi
2. Install dependencies:
    ```
    pnpm install
    or
    npm install
    ```
3. Start the development server:
    ```
    pnpm dev
    or
    npm run dev
    ```
4. Open http://localhost:5173 in your browser.

## Usage:
1. Text Mode: Type your message and press Enter to chat with VIXI.
2. Call Mode: Click the call button to start a voice conversation. Speak 
    into your microphone and VIXI will respond with synthesized speech.
3. Language Selection: Choose your preferred language from the language selector.

## Technologies Used
- SvelteKit
- Vite
- Tailwind CSS
- Puter AI (for text-to-speech)
- Groq (for AI chat and transcription)
## License
MIT