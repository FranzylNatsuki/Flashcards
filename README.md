# Agentic Vue Flashcards

A lightweight, clean desktop flashcard application built with Vue 3, Vite, and Electron. 

## Features
- **JSON Powered**: Empty by default. Just upload a `.json` file containing an array of `{ term: "", definition: "" }` objects to instantly load your deck. Perfect for having AI generate study materials for you.
- **Phased Reviewing**: Cards you mark as "Not Memorized" are kept in a bucket. Once you clear the current deck, the app automatically shuffles your missed cards and starts the next phase until you've mastered them all.
- **Searchable Dictionary**: Click "View All Cards" to open a 3-column, responsive grid showing every card in the deck, complete with a real-time search bar.
- **End-of-Session Analytics**: When you finish a deck, the app displays a grid of any terms you struggled with (missed 2 or more times) so you know exactly what to focus on.

## Json Formatting 

```json
[
  {
    "id": 1,
    "term": "",
    "definition": ""
  }
]
```

## Development Setup

To run the app in development mode (with hot-reloading):

```bash
npm install
npm run dev
```

## Building the Desktop App

This project uses `electron-builder` to package the web code into standalone native desktop executables. The output files will be placed in the `release/` folder.

### Build for Windows (.exe)
*Note: It is highly recommended to run this command on a Windows machine.*
```bash
npm run build -- --win
```

### Build for macOS (.dmg)
*Note: You must run this command on an actual Mac to properly package macOS applications.*
```bash
npm run build -- --mac
```

### Build for Linux (.AppImage)
*Note: Run this on a Linux machine.*
```bash
npm run build -- --linux
```

### Build for your current OS (Default)
Running the build command without any flags will automatically build the app for the operating system you are currently using:
```bash
npm run build
```
