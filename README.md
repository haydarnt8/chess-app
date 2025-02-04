# Nine Kings Chess Puzzle

An interactive chess puzzle implementation where the goal is to place 9 kings on a 9x9 chessboard without any king being able to attack another king.

## Features

- 🎮 Interactive 9x9 chessboard
- 👑 Place and move kings with drag-and-drop or click
- 🎯 Visual indicators for possible moves
- ⚡ Real-time conflict detection
- 🎲 Auto-generate valid solutions
- 💫 Smooth animations and visual feedback
- 📱 Responsive design for all devices

## How to Play

1. Click "Generate New Solution" for a random valid arrangement
2. Move kings by:
   - Dragging and dropping
   - Clicking a king and then clicking a valid destination
3. Valid moves are:
   - One square in any direction
   - Cannot move to a square where kings would attack each other

## Technical Details

### Built With
- Vue 3 (Composition API)
- CSS3 with advanced animations
- SVG for chess pieces

### Key Components
- `ChessBoard.vue`: Main component handling the game logic and UI

### Game Rules
- Board size: 9x9
- Number of kings: 9
- Kings attack pattern: One square in any direction
- Goal: No king should be able to attack another king

## Development

### Prerequisites
```bash
node.js >= 14.0.0
npm or yarn
```

### Installation
```bash
# Clone the repository
git clone [repository-url]

# Install dependencies
npm install

# Start development server
npm run dev
```

### Project Structure
```
chess-app/
├── src/
│   ├── components/
│   │   └── ChessBoard.vue
│   ├── App.vue
│   └── main.js
├── public/
└── package.json
```

## Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

MIT License - See LICENSE file for details

## Acknowledgments

- Chess piece SVG designs
- Vue.js team for the fantastic framework
- Chess puzzle community for inspiration
