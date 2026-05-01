# GEMINI.md - Text to Speech Vue 3 Project

## Project Overview
This project is a modern, responsive Text-to-Speech (TTS) application built with **Vue 3** and **Vite**. It leverages the browser's native **Web Speech API** to convert user-inputted text into synthesized speech.

### Main Technologies
- **Vue 3**: Frontend framework using the Composition API and `<script setup>` syntax.
- **Vite**: Modern build tool and development server.
- **Web Speech API**: Browser API for speech synthesis.
- **Vanilla CSS**: Custom styling with CSS variables and responsive design principles.

### Architecture
- **`index.html`**: Entry point for the application.
- **`src/main.js`**: Initializes the Vue application and mounts the root component.
- **`src/App.vue`**: The main application component containing the template and core business logic.
- **`src/style.css`**: Global stylesheet containing all application styles.

## Building and Running

### Development
To start the development server with hot-reload:
```bash
npm run dev
```

### Production Build
To build the application for production:
```bash
npm run build
```

### Preview
To preview the production build locally:
```bash
npm run preview
```

## Development Conventions

### Coding Style
- **Vue 3 Composition API**: Use `<script setup>` for component logic.
- **Reactive State**: Use `ref` and `computed` for state management within components.
- **Naming**: Follow standard JavaScript (camelCase) and Vue (PascalCase for components, kebab-case for templates) conventions.
- **Styling**: Prefer custom CSS variables for consistent theming and layout management.

### Features & Logic
- **Voice Selection**: Dynamically loads and filters available system voices.
- **Speech Controls**: Provides granular control over rate, pitch, and volume.
- **Visual Feedback**: Includes a CSS-based waveform animation and status indicators for a better user experience.
- **Localization**: Initial focus on Bulgarian language support, but extensible to all languages supported by the Web Speech API.
