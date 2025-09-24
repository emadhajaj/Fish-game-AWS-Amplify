# Fish-game-AWS-Amplify

# 1. AWS Amplify Console Steps


1. **Create New App**:
    - Click "New app" → "Host web app"
    - Choose your Git provider (GitHub/GitLab/Bitbucket)
    - Authorize AWS Amplify to access your repositories
    
2. **Configure Repository**:
    - Select your repository containing the fish game
    - Choose the branch (usually 'main' or 'master')
    
3. **Build Settings**:
    - App name: "Fish-game-AWS-Amplify" 
    - For a simple HTML/JS game, use these build settings:
    
    ```yaml
    version: 1
	frontend:
	phases:
	 preBuild:
	   commands:
	     - echo "No preBuild step needed"
	 build:
	   commands:
	     - echo "No build required for static HTML game"
	artifacts:
	 baseDirectory: .
	 files:
	   - '**/*'
	cache:
	 paths: []
    ```


# 2. Build The Game in the repo

## `amplify.yml` 


```yaml
version: 1
frontend:
  phases:
    build:
      commands:
        - echo "Building static HTML game..."
  artifacts:
    baseDirectory: /
    files:
      - '**/*'
  cache:
    paths: []
```


## `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fish Game - Mouse Controlled</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background: linear-gradient(180deg, #87CEEB 0%, #4682B4 50%, #191970 100%);
            font-family: Arial, sans-serif;
            overflow: hidden;
            cursor: none;
        }

        canvas {
            display: block;
            background: linear-gradient(180deg, 
                rgba(135, 206, 235, 0.8) 0%, 
                rgba(70, 130, 180, 0.9) 50%, 
                rgba(25, 25, 112, 1) 100%);
        }

        .ui {
            position: absolute;
            top: 20px;
            left: 20px;
            color: white;
            font-size: 18px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.7);
            z-index: 100;
        }

        .game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0,0,0,0.8);
            color: white;
            padding: 30px;
            border-radius: 10px;
            text-align: center;
            display: none;
            z-index: 200;
        }

        .restart-btn {
            background: #4CAF50;
            color: white;
            border: none;
            padding: 10px 20px;
            font-size: 16px;
            border-radius: 5px;
            cursor: pointer;
            margin-top: 15px;
        }

        .restart-btn:hover {
            background: #45a049;
        }

        .bubbles {
            position: absolute;
            width: 100%;
            height: 100%;
            pointer-events: none;
            z-index: 1;
        }

        .bubble {
            position: absolute;
            background: rgba(255,255,255,0.1);
            border-radius: 50%;
            animation: float 3s infinite ease-in-out;
        }

        @keyframes float {
            0% { transform: translateY(100vh) scale(0); opacity: 1; }
            100% { transform: translateY(-100px) scale(1); opacity: 0; }
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>
    
    <div class="ui">
        <div>Score: <span id="score">0</span></div>
        <div>Lives: <span id="lives">3</span></div>
        <div>Level: <span id="level">1</span></div>
    </div>

    <div class="game-over" id="gameOver">
        <h2>Game Over!</h2>
        <p>Final Score: <span id="finalScore">0</span></p>
        <button class="restart-btn" onclick="restartGame()">Play Again</button>
    </div>

    <div class="bubbles" id="bubbles"></div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        
        // Set canvas size
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        // Game variables
        let score = 0;
        let lives = 3;
        let level = 1;
        let gameRunning = true;
        let mouseX = canvas.width / 2;
        let mouseY = canvas.height / 2;

        // Player fish
        const player = {
            x: canvas.width / 2,
            y: canvas.height / 2,
            size: 30,
            speed: 0.1,
            color: '#FF6B35'
        };

        // Arrays for game objects
        let enemies = [];
        let food = [];
        let particles = [];

        // Mouse tracking
        canvas.addEventListener('mousemove', (e) => {
            mouseX = e.clientX;
            mouseY = e.clientY;
        });

        // Enemy fish class
        class Enemy {
            constructor() {
                this.size = Math.random() * 40 + 20;
                this.speed = Math.random() * 2 + 1;
                this.x = Math.random() < 0.5 ? -this.size : canvas.width + this.size;
                this.y = Math.random() * (canvas.height - 100) + 50;
                this.color = this.getRandomColor();
                this.direction = this.x < 0 ? 1 : -1;
            }

            getRandomColor() {
                const colors = ['#8B0000', '#FF4500', '#B22222', '#DC143C', '#800080'];
                return colors[Math.floor(Math.random() * colors.length)];
            }

            update() {
                this.x += this.speed * this.direction;
                
                // Remove if off screen
                if (this.direction > 0 && this.x > canvas.width + this.size) {
                    return false;
                }
                if (this.direction < 0 && this.x < -this.size) {
                    return false;
                }
                return true;
            }

            draw() {
                ctx.fillStyle = this.color;
                ctx.beginPath();
                
                // Fish body
                ctx.ellipse(this.x, this.y, this.size, this.size * 0.6, 0, 0, Math.PI * 2);
                ctx.fill();
                
                // Fish tail
                const tailSize = this.size * 0.5;
                ctx.beginPath();
                if (this.direction > 0) {
                    ctx.moveTo(this.x - this.size, this.y);
                    ctx.lineTo(this.x - this.size - tailSize, this.y - tailSize * 0.5);
                    ctx.lineTo(this.x - this.size - tailSize, this.y + tailSize * 0.5);
                } else {
                    ctx.moveTo(this.x + this.size, this.y);
                    ctx.lineTo(this.x + this.size + tailSize, this.y - tailSize * 0.5);
                    ctx.lineTo(this.x + this.size + tailSize, this.y + tailSize * 0.5);
                }
                ctx.closePath();
                ctx.fill();

                // Fish eye
                ctx.fillStyle = 'white';
                ctx.beginPath();
                const eyeX = this.direction > 0 ? this.x + this.size * 0.3 : this.x - this.size * 0.3;
                ctx.arc(eyeX, this.y - this.size * 0.2, this.size * 0.15, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.fillStyle = 'black';
                ctx.beginPath();
                ctx.arc(eyeX, this.y - this.size * 0.2, this.size * 0.08, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        // Food class
        class Food {
            constructor() {
                this.x = Math.random() * canvas.width;
                this.y = Math.random() * canvas.height;
                this.size = Math.random() * 8 + 4;
                this.color = '#00FF00';
                this.life = 300; // frames
            }

            update() {
                this.life--;
                return this.life > 0;
            }

            draw() {
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
                ctx.fill();
                
                // Add sparkle effect
                ctx.fillStyle = 'rgba(255,255,255,0.8)';
                ctx.beginPath();
                ctx.arc(this.x - this.size * 0.3, this.y - this.size * 0.3, this.size * 0.2, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        // Particle class for effects
        class Particle {
            constructor(x, y, color) {
                this.x = x;
                this.y = y;
                this.vx = (Math.random() - 0.5) * 4;
                this.vy = (Math.random() - 0.5) * 4;
                this.size = Math.random() * 4 + 2;
                this.color = color;
                this.life = 30;
            }

            update() {
                this.x += this.vx;
                this.y += this.vy;
                this.vx *= 0.98;
                this.vy *= 0.98;
                this.size *= 0.95;
                this.life--;
                return this.life > 0;
            }

            draw() {
                ctx.fillStyle = this.color;
                ctx.globalAlpha = this.life / 30;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
                ctx.fill();
                ctx.globalAlpha = 1;
            }
        }

        // Draw player fish
        function drawPlayer() {
            // Smooth movement towards mouse
            player.x += (mouseX - player.x) * player.speed;
            player.y += (mouseY - player.y) * player.speed;

            // Determine direction based on movement
            const dx = mouseX - player.x;
            const facingRight = dx > 0;

            ctx.fillStyle = player.color;
            ctx.beginPath();
            
            // Fish body
            ctx.ellipse(player.x, player.y, player.size, player.size * 0.6, 0, 0, Math.PI * 2);
            ctx.fill();
            
            // Fish tail
            const tailSize = player.size * 0.5;
            ctx.beginPath();
            if (facingRight) {
                ctx.moveTo(player.x - player.size, player.y);
                ctx.lineTo(player.x - player.size - tailSize, player.y - tailSize * 0.5);
                ctx.lineTo(player.x - player.size - tailSize, player.y + tailSize * 0.5);
            } else {
                ctx.moveTo(player.x + player.size, player.y);
                ctx.lineTo(player.x + player.size + tailSize, player.y - tailSize * 0.5);
                ctx.lineTo(player.x + player.size + tailSize, player.y + tailSize * 0.5);
            }
            ctx.closePath();
            ctx.fill();

            // Fish eye
            ctx.fillStyle = 'white';
            ctx.beginPath();
            const eyeX = facingRight ? player.x + player.size * 0.3 : player.x - player.size * 0.3;
            ctx.arc(eyeX, player.y - player.size * 0.2, player.size * 0.15, 0, Math.PI * 2);
            ctx.fill();
            
            ctx.fillStyle = 'black';
            ctx.beginPath();
            ctx.arc(eyeX, player.y - player.size * 0.2, player.size * 0.08, 0, Math.PI * 2);
            ctx.fill();
        }

        // Collision detection
        function checkCollision(obj1, obj2) {
            const dx = obj1.x - obj2.x;
            const dy = obj1.y - obj2.y;
            const distance = Math.sqrt(dx * dx + dy * dy);
            return distance < (obj1.size + obj2.size);
        }

        // Create bubbles
        function createBubble() {
            const bubble = document.createElement('div');
            bubble.classList.add('bubble');
            bubble.style.left = Math.random() * window.innerWidth + 'px';
            bubble.style.width = bubble.style.height = Math.random() * 20 + 10 + 'px';
            bubble.style.animationDelay = Math.random() * 3 + 's';
            document.getElementById('bubbles').appendChild(bubble);

            setTimeout(() => {
                bubble.remove();
            }, 3000);
        }

        // Game loop
        function gameLoop() {
            if (!gameRunning) return;

            // Clear canvas
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw player
            drawPlayer();

            // Spawn enemies
            if (Math.random() < 0.02 + level * 0.005) {
                enemies.push(new Enemy());
            }

            // Spawn food
            if (Math.random() < 0.01) {
                food.push(new Food());
            }

            // Update and draw enemies
            enemies = enemies.filter(enemy => {
                const alive = enemy.update();
                if (alive) {
                    enemy.draw();
                    
                    // Check collision with player
                    if (checkCollision(player, enemy)) {
                        lives--;
                        // Add damage particles
                        for (let i = 0; i < 10; i++) {
                            particles.push(new Particle(player.x, player.y, '#FF0000'));
                        }
                        return false; // Remove enemy
                    }
                }
                return alive;
            });

            // Update and draw food
            food = food.filter(foodItem => {
                const alive = foodItem.update();
                if (alive) {
                    foodItem.draw();
                    
                    // Check collision with player
                    if (checkCollision(player, foodItem)) {
                        score += 10;
                        // Add sparkle particles
                        for (let i = 0; i < 5; i++) {
                            particles.push(new Particle(foodItem.x, foodItem.y, '#00FF00'));
                        }
                        return false; // Remove food
                    }
                }
                return alive;
            });

            // Update and draw particles
            particles = particles.filter(particle => {
                const alive = particle.update();
                if (alive) {
                    particle.draw();
                }
                return alive;
            });

            // Create bubbles occasionally
            if (Math.random() < 0.05) {
                createBubble();
            }

            // Level progression
            if (score > level * 100) {
                level++;
            }

            // Update UI
            document.getElementById('score').textContent = score;
            document.getElementById('lives').textContent = lives;
            document.getElementById('level').textContent = level;

            // Check game over
            if (lives <= 0) {
                gameOver();
                return;
            }

            requestAnimationFrame(gameLoop);
        }

        // Game over function
        function gameOver() {
            gameRunning = false;
            document.getElementById('finalScore').textContent = score;
            document.getElementById('gameOver').style.display = 'block';
        }

        // Restart game
        function restartGame() {
            score = 0;
            lives = 3;
            level = 1;
            enemies = [];
            food = [];
            particles = [];
            gameRunning = true;
            player.x = canvas.width / 2;
            player.y = canvas.height / 2;
            document.getElementById('gameOver').style.display = 'none';
            gameLoop();
        }

        // Handle window resize
        window.addEventListener('resize', () => {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        });

        // Start the game
        gameLoop();
    </script>
</body>
</html>
```


# 3. Deployment 

Right now the website is working, but for production

1. **Domain Setup**: 
	After deployment, you can add a custom domain in Amplify console under "Domain management"
    
2. **Environment Variables**: 
	If you need any config variables, add them in Amplify Console → App Settings → Environment Variables
    
3. **Branch Management**: 
	You can set up multiple environments (dev, staging, prod) by connecting different branches
    
4. **Monitoring**: Check the build logs in Amplify Console if deployment fails
5. **SSL**: Amplify automatically provides SSL certificates for your domain




# How to Play

- Move your mouse to control the fish
- Eat green food to gain points
- Avoid red enemy fish
- Survive as long as possible!
