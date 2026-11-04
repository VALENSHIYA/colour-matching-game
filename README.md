# colour-matching-game
A fun colour matching game built with HTML, CSS and JavaScript.


<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>🌈 Colour Matching Game</title>

<style>
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    padding: 15px;
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    font-family: Arial, sans-serif;
    background: linear-gradient(135deg, #b9eaff, #ffe0f0);
}

.game {
    width: 100%;
    max-width: 850px;
    text-align: center;
}

h1 {
    color: #7535b5;
    font-size: clamp(25px, 5vw, 38px);
    margin: 5px 0 15px;
}

.info {
    display: flex;
    justify-content: space-between;
    align-items: center;
    flex-wrap: wrap;
    gap: 10px;
    font-size: clamp(14px, 2.5vw, 19px);
    font-weight: bold;
    margin-bottom: 12px;
}

#level { color: #7135bd; }
#score { color: #07833a; }
#timer { color: #d62828; }
#lives { font-size: 23px; letter-spacing: 2px; }

.progress-bg {
    width: 100%;
    height: 13px;
    background: #ffffffaa;
    border-radius: 20px;
    overflow: hidden;
    margin-bottom: 12px;
}

#progress {
    height: 100%;
    width: 0%;
    background: linear-gradient(90deg, #20bf55, #a8e063);
    transition: width 0.25s;
}

canvas {
    display: block;
    width: 100%;
    height: auto;
    border: 4px solid white;
    border-radius: 18px;
    box-shadow: 0 8px 24px #62457a35;
    touch-action: manipulation;
}

#message {
    min-height: 42px;
    padding: 10px 2px;
    font-size: 18px;
    font-weight: bold;
    color: #7434aa;
}

button {
    border: none;
    border-radius: 25px;
    padding: 12px 27px;
    background: #843bc8;
    color: white;
    font-size: 17px;
    font-weight: bold;
    cursor: pointer;
    box-shadow: 0 4px 0 #59248b;
}

button:hover {
    background: #a259df;
}

button:active {
    transform: translateY(2px);
    box-shadow: 0 2px 0 #59248b;
}

.help {
    color: #554768;
    font-size: 14px;
    margin-top: 13px;
}

.heart-break {
    animation: shake 0.35s;
}

@keyframes shake {
    0%, 100% { transform: translateX(0); }
    25% { transform: translateX(-5px); }
    75% { transform: translateX(5px); }
}
</style>
</head>

<body>
<div class="game">

    <h1>🌈 Colour Matching Game 🎈</h1>

    <div class="info">
        <span id="level">Level 1: Easy</span>
        <span id="score">🏆 Score: 0</span>
        <span id="timer">⏱️ Time: 60</span>
        <span id="lives" aria-label="3 lives">❤️ ❤️ ❤️</span>
    </div>

    <div class="progress-bg">
        <div id="progress"></div>
    </div>

    <canvas id="game" width="800" height="600"></canvas>

    <div id="message" aria-live="polite">
        Find the balloon matching the colour!
    </div>

    <button id="restart">🔄 Restart Game</button>

    <div class="help">
        Get 5 correct answers per level. You have 3 hearts!
    </div>
</div>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const levelLabel = document.getElementById("level");
const scoreLabel = document.getElementById("score");
const timerLabel = document.getElementById("timer");
const livesLabel = document.getElementById("lives");
const progressBar = document.getElementById("progress");
const message = document.getElementById("message");

const colours = {
    Red: "#ff3030",
    Green: "#00b84a",
    Blue: "#087cff",
    Yellow: "#e4ad00",
    Purple: "#b400ff",
    Orange: "#ff8c00"
};

const rainbow = [
    "#ff0000", "#ff8500", "#ffff00",
    "#00c800", "#0064ff", "#4b0082", "#9400d3"
];

const positions = [
    [150, 195],
    [400, 195],
    [650, 195],
    [150, 425],
    [400, 425],
    [650, 425]
];

// Difficulty settings for each level
const levels = [
    {
        name: "Easy",
        duration: 60,
        balloonSize: 35,
        speed: 0.8,
        goal: 5
    },
    {
        name: "Medium",
        duration: 45,
        balloonSize: 29,
        speed: 1.7,
        goal: 5
    },
    {
        name: "Hard",
        duration: 30,
        balloonSize: 23,
        speed: 2.8,
        goal: 5
    }
];

let levelIndex = 0;
let score = 0;
let levelScore = 0;
let lives = 3;
const maxLives = 3;

let remaining = 60;
let balloons = [];
let target = "";

let startTime = 0;
let gameOver = false;
let gameWon = false;

let feedbackUntil = 0;
let animationId;
let hitFlashUntil = 0;
let hitFlashColour = "#ffffff";

// Shuffle an array randomly
function shuffle(array) {
    for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
}

// Create six balloons
function createBalloons() {
    const names = shuffle(Object.keys(colours));

    balloons = names.map((name, i) => ({
        name: name,
        colour: colours[name],
        x: positions[i][0],
        baseY: positions[i][1],
        phase: Math.random() * Math.PI * 2
    }));

    target = balloons[
        Math.floor(Math.random() * balloons.length)
    ].name;
}

// Update score, level, lives and progress
function updateInfo() {
    const level = levels[levelIndex];

    levelLabel.textContent =
        `Level ${levelIndex + 1}: ${level.name}`;

    scoreLabel.textContent = `🏆 Score: ${score}`;

    livesLabel.textContent =
        Array.from({ length: maxLives }, (_, i) =>
            i < lives ? "❤️" : "🖤"
        ).join(" ");

    livesLabel.setAttribute(
        "aria-label",
        `${lives} of ${maxLives} lives remaining`
    );

    progressBar.style.width =
        (levelScore / level.goal * 100) + "%";
}

// Display feedback to the player
function showMessage(text, colour = "#7434aa") {
    message.textContent = text;
    message.style.color = colour;
    feedbackUntil = performance.now() + 1600;
}

// Background clouds
function drawCloud(x, y) {
    ctx.fillStyle = "white";
    ctx.beginPath();
    ctx.arc(x, y, 24, 0, Math.PI * 2);
    ctx.arc(x + 30, y - 15, 34, 0, Math.PI * 2);
    ctx.arc(x + 65, y, 24, 0, Math.PI * 2);
    ctx.fill();
    ctx.fillRect(x - 5, y, 75, 24);
}

// Stars
function drawStar(x, y, size) {
    ctx.beginPath();

    for (let i = 0; i < 10; i++) {
        const angle = i * Math.PI / 5 - Math.PI / 2;
        const radius = i % 2 === 0 ? size : size / 2;
        const px = x + Math.cos(angle) * radius;
        const py = y + Math.sin(angle) * radius;

        if (i === 0) {
            ctx.moveTo(px, py);
        } else {
            ctx.lineTo(px, py);
        }
    }

    ctx.closePath();
    ctx.fillStyle = "#ffd700";
    ctx.fill();
}

// Rainbow
function drawRainbow() {
    rainbow.forEach((colour, i) => {
        ctx.beginPath();
        ctx.arc(
            400, 540, 295 - i * 12,
            Math.PI, Math.PI * 2
        );
        ctx.strokeStyle = colour;
        ctx.lineWidth = 12;
        ctx.stroke();
    });
}

// Sky and scenery
function drawBackground() {
    const gradient = ctx.createLinearGradient(0, 0, 0, 600);
    gradient.addColorStop(0, "#87cefa");
    gradient.addColorStop(1, "#e5f8ff");

    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, 800, 600);

    drawCloud(70, 110);
    drawCloud(600, 100);

    drawStar(100, 50, 12);
    drawStar(700, 50, 12);
    drawStar(350, 90, 10);
    drawStar(500, 60, 10);

    drawRainbow();
}

// Calculate the current balloon position
function balloonY(balloon, now) {
    const settings = levels[levelIndex];

    return balloon.baseY +
        Math.sin(
            now / (600 / settings.speed) + balloon.phase
        ) * (10 + settings.speed * 3);
}

// Draw a balloon and its string
function drawBalloon(balloon, now) {
    const size = levels[levelIndex].balloonSize;
    const y = balloonY(balloon, now);

    // String
    ctx.beginPath();
    ctx.moveTo(balloon.x, y + size * 0.9);
    ctx.lineTo(balloon.x, y + size * 2.3);
    ctx.strokeStyle = "#777777";
    ctx.lineWidth = 2;
    ctx.stroke();

    // Balloon body
    ctx.beginPath();
    ctx.ellipse(
        balloon.x, y,
        size * 0.72, size,
        0, 0, Math.PI * 2
    );

    ctx.fillStyle = balloon.colour;
    ctx.fill();

    ctx.strokeStyle = "#ffffff88";
    ctx.lineWidth = 2;
    ctx.stroke();

    // Knot
    ctx.beginPath();
    ctx.moveTo(balloon.x - 5, y + size * 0.85);
    ctx.lineTo(balloon.x + 5, y + size * 0.85);
    ctx.lineTo(balloon.x, y + size * 1.08);
    ctx.closePath();
    ctx.fillStyle = balloon.colour;
    ctx.fill();

    // Balloon shine
    ctx.beginPath();
    ctx.ellipse(
        balloon.x - size * 0.25,
        y - size * 0.32,
        size * 0.12,
        size * 0.25,
        -0.4, 0, Math.PI * 2
    );
    ctx.fillStyle = "#ffffffcc";
    ctx.fill();

    // Flash the screen briefly after a mistake
    if (performance.now() < hitFlashUntil) {
        ctx.beginPath();
        ctx.arc(
            balloon.x, y,
            size * 1.25,
            0, Math.PI * 2
        );
        ctx.strokeStyle = hitFlashColour;
        ctx.lineWidth = 4;
        ctx.stroke();
    }
}

// Target colour heading
function drawHeading() {
    ctx.textAlign = "center";
    ctx.font = "bold 32px Arial";

    const heading = `Find the ${target.toUpperCase()} Balloon!`;

    ctx.fillStyle = "#ffffff";
    ctx.fillText(heading, 400, 42);

    ctx.fillStyle = colours[target];
    ctx.fillText(heading, 398, 40);
}

// Game-over and victory screen
function drawEndScreen() {
    ctx.fillStyle = "rgba(255,230,240,0.97)";
    ctx.fillRect(0, 0, 800, 600);

    drawRainbow();

    ctx.textAlign = "center";

    ctx.font = "bold 48px Arial";
    ctx.fillStyle = gameWon ? "#7535b5" : "#e34269";

    ctx.fillText(
        gameWon ? "🎉 You Won! 🎉" : "💔 Game Over!",
        400, 220
    );

    ctx.font = "bold 38px Arial";
    ctx.fillStyle = "#6036a4";
    ctx.fillText("Final Score: " + score, 400, 285);

    ctx.font = "26px Arial";
    ctx.fillStyle = "#444444";

    ctx.fillText(
        gameWon
            ? "You completed all 3 levels!"
            : lives === 0
                ? "You lost all 3 hearts!"
                : "Time ran out!",
        400, 340
    );

    ctx.font = "22px Arial";
    ctx.fillText("Press Restart Game to play again", 400, 390);

    drawStar(250, 280, 18);
    drawStar(550, 280, 18);
}

// Move to the next level
function advanceLevel() {
    if (levelIndex >= levels.length - 1) {
        gameWon = true;
        gameOver = true;
        showMessage("Amazing! All levels completed! 🏆", "#16883c");
        return;
    }

    levelIndex++;
    levelScore = 0;
    startTime = performance.now();
    remaining = levels[levelIndex].duration;

    createBalloons();
    updateInfo();

    timerLabel.textContent = "⏱️ Time: " + remaining;

    showMessage(
        `🎉 Level ${levelIndex + 1}: ${levels[levelIndex].name}!`,
        "#7535b5"
    );
}

// Main animation loop
function draw(now) {
    drawBackground();

    if (!gameOver) {
        const level = levels[levelIndex];

        remaining = Math.max(
            0,
            level.duration - Math.floor((now - startTime) / 1000)
        );

        timerLabel.textContent = "⏱️ Time: " + remaining;
        timerLabel.style.color =
            remaining <= 10 ? "#ff0000" : "#d62828";

        if (remaining === 0) {
            gameOver = true;
            gameWon = false;
        }
    }

    if (gameOver) {
        drawEndScreen();
        return;
    }

    drawHeading();

    balloons.forEach(balloon => drawBalloon(balloon, now));

    if (now > feedbackUntil) {
        message.textContent = "";
    }

    animationId = requestAnimationFrame(draw);
}

// Balloon click handling
canvas.addEventListener("click", event => {
    if (gameOver) return;

    const rect = canvas.getBoundingClientRect();

    const mx =
        (event.clientX - rect.left) * canvas.width / rect.width;

    const my =
        (event.clientY - rect.top) * canvas.height / rect.height;

    const now = performance.now();
    const size = levels[levelIndex].balloonSize;

    for (const balloon of balloons) {
        const y = balloonY(balloon, now);

        const dx = mx - balloon.x;
        const dy = my - y;

        const rx = size * 0.85;
        const ry = size * 1.12;

        // Only clicks on the balloon body count
        if (
            (dx * dx) / (rx * rx) +
            (dy * dy) / (ry * ry) <= 1
        ) {
            if (balloon.name === target) {
                // Correct answer
                score++;
                levelScore++;

                updateInfo();

                showMessage("Correct! 🎉 +1 point", "#16883c");

                if (levelScore >= levels[levelIndex].goal) {
                    advanceLevel();
                } else {
                    createBalloons();
                }

            } else {
                // Wrong answer: lose one heart
                lives--;

                hitFlashUntil = performance.now() + 350;
                hitFlashColour = "#ff3030";

                updateInfo();

                if (lives <= 0) {
                    lives = 0;
                    updateInfo();

                    gameOver = true;
                    gameWon = false;

                    showMessage("💔 All hearts lost!", "#e34269");
                } else {
                    showMessage(
                        "💔 Wrong colour! " + lives +
                        (lives === 1 ? " heart left!" : " hearts left!"),
                        "#e05a28"
                    );
                }
            }

            break;
        }
    }
});

// Restart everything
function startGame() {
    cancelAnimationFrame(animationId);

    levelIndex = 0;
    score = 0;
    levelScore = 0;
    lives = maxLives;
    remaining = levels[0].duration;

    gameOver = false;
    gameWon = false;

    startTime = performance.now();
    feedbackUntil = 0;
    hitFlashUntil = 0;

    createBalloons();
    updateInfo();

    timerLabel.textContent = "⏱️ Time: 60";
    timerLabel.style.color = "#d62828";

    showMessage("Let's play! Find the matching balloon 🎈");

    animationId = requestAnimationFrame(draw);
}

document.getElementById("restart").addEventListener("click", startGame);

document.addEventListener("keydown", event => {
    if (event.key.toLowerCase() === "r") {
        startGame();
    }
});

startGame();
</script>
</body>
</html>
