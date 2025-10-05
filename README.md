# Soccer-Mobile-and-PC
Đây là bản thử của Soccer Mobile and PC
<!doctype html>
<html lang="vi">
<head>
  <meta charset="utf-8" />
  <title>Soccer Demo — SoccerMobileAndPC</title>
  <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no" />
  <style>
    html,body { margin:0; padding:0; height:100%; background:#0b6623; -webkit-user-select:none; user-select:none; }
    #gameContainer { width:100%; height:100vh; position:relative; overflow:hidden; }
    /* UI */
    .hud {
      position:absolute; left:50%; transform:translateX(-50%); top:8px; z-index:60;
      background: rgba(0,0,0,0.35); color:#fff; padding:8px 14px; border-radius:10px; font-weight:700;
      display:flex; gap:12px; align-items:center; font-family: Arial, sans-serif;
    }
    #goalText {
      position:absolute; left:50%; transform:translateX(-50%); top:40%; z-index:70;
      color:#fff; font-size:34px; font-weight:900; text-shadow:0 3px 8px rgba(0,0,0,0.6);
      display:none;
    }
    /* Touch controls */
    .touch-left {
      position:absolute; left:14px; bottom:14px; z-index:70;
      width:120px; height:120px; border-radius:999px; background:rgba(255,255,255,0.06); display:flex; align-items:center; justify-content:center;
    }
    .stick {
      width:56px; height:56px; border-radius:999px; background:rgba(255,255,255,0.16);
      touch-action:none;
    }
    .touch-right {
      position:absolute; right:14px; bottom:22px; z-index:70; display:flex; flex-direction:column; gap:12px; align-items:center;
    }
    .btn {
      width:84px; height:48px; border-radius:10px; background:rgba(255,255,255,0.12); color:#fff; display:flex; align-items:center; justify-content:center; font-weight:800;
      user-select:none;
    }
    .small { font-size:12px; opacity:0.9; }
    /* overlay */
    #overlay {
      position:absolute; inset:0; display:flex; align-items:center; justify-content:center; z-index:90; background:rgba(0,0,0,0.6); color:#fff; flex-direction:column; gap:10px; font-family: Arial, sans-serif;
    }
    .green { background:#2ecc71; color:#063; padding:10px 14px; border-radius:8px; font-weight:800; }
  </style>
</head>
<body>
  <div id="gameContainer"></div>

  <div class="hud" id="hud">
    <div id="score">VN 0 - 0 TH</div>
    <div id="time">120s</div>
  </div>

  <div id="goalText">GOAL!</div>

  <!-- Touch UI -->
  <div class="touch-left" id="joyOuter" style="display:none;">
    <div class="stick" id="stick"></div>
  </div>

  <div class="touch-right" id="touchBtns" style="display:none;">
    <div class="btn" id="shootBtn">SÚT</div>
    <div class="small">Điều khiển: cần ảo + nút SÚT</div>
  </div>

  <!-- overlay start -->
  <div id="overlay">
    <div style="font-size:22px; font-weight:900;">Soccer Demo — VN vs TH</div>
    <div class="small">Top-down, mobile controls (joystick + shoot). 2 phút trận.</div>
    <div style="display:flex; gap:8px;">
      <button id="playBtn" class="green">Chơi ngay</button>
      <button id="howBtn" class="btn">Hướng dẫn</button>
    </div>
    <div class="small">Nếu chạy trên máy tính, dùng phím mũi tên + Space</div>
  </div>

  <!-- Phaser 3 -->
  <script src="https://cdn.jsdelivr.net/npm/phaser@3.60.0/dist/phaser.min.js"></script>
  <script>
  // --- CONFIG ---
  const config = {
    type: Phaser.AUTO,
    parent: 'gameContainer',
    width: Math.min(window.innerWidth, 900),
    height: Math.min(window.innerHeight, 680),
    backgroundColor: '#2a7b2a',
    physics: {
      default: 'arcade',
      arcade: { gravity: { y: 0 }, debug: false }
    },
    scale: {
      mode: Phaser.Scale.FIT,
      autoCenter: Phaser.Scale.CENTER_BOTH
    },
    scene: { preload, create, update }
  };

  let game = new Phaser.Game(config);

  // --- STATE ---
  let playerMain, ball;
  let cursors;
  let joystick = { active:false, startX:0, startY:0, dx:0, dy:0 };
  let shootReady = true;
  let scoreA = 0, scoreB = 0;
  let matchTime = 120;
  let timerEvent;
  let goalTextEl, hudEl;
  const teamColours = { a:0xf39c12, b:0x3498db };
  const playerGroupA = [], playerGroupB = [];
  let isMobile = /Mobi|Android/i.test(navigator.userAgent);

  function preload() {
    // none: we'll use graphics
    this.load.audio('crowd', 'https://cdn.jsdelivr.net/gh/andrewguc/short-assets@main/crowd_short.ogg').on('loaderror', ()=>{}); // optional CDN fallback; silent if fails
    this.load.audio('kick', 'https://cdn.jsdelivr.net/gh/andrewguc/short-assets@main/kick_short.ogg').on('loaderror', ()=>{});
  }

  function create() {
    const scene = this;
    const w = this.scale.width, h = this.scale.height;

    // Field
    const g = this.add.graphics();
    g.fillStyle(0x1f8b4c, 1);
    g.fillRect(0,0,w,h);
    g.lineStyle(4, 0xffffff, 0.8);
    g.strokeRect(24, 24, w-48, h-48);
    g.strokeLineShape(new Phaser.Geom.Line(w/2, 24, w/2, h-24));
    g.strokeCircle(w/2, h/2, Math.min(w,h)*0.12);

    // Goals (visual)
    const goalH = 140;
    g.fillStyle(0x0b3b1f, 1);
    g.fillRect(24 - 6, h/2 - goalH/2, 12, goalH);
    g.fillRect(w-24 - 6, h/2 - goalH/2, 12, goalH);

    // Create ball
    ball = this.add.circle(w/2, h/2, 10, 0xffffff);
    this.physics.add.existing(ball);
    ball.body.setCircle(10);
    ball.body.setBounce(0.98);
    ball.body.setCollideWorldBounds(true);

    // Player spawn positions (3 + GK each side)
    const leftX = w*0.22, rightX = w*0.78;
    const rows = [-80, 0, 80];
    for (let i=0;i<3;i++){
      const p = createPlayer(scene, leftX, h/2 + rows[i], teamColours.a, 'A'+i);
      playerGroupA.push(p);
    }
    // GK left
    const gkA = createPlayer(scene, leftX - 80, h/2, teamColours.a, 'A_GK', true);
    playerGroupA.push(gkA);

    for (let i=0;i<3;i++){
      const p = createPlayer(scene, rightX, h/2 + rows[i], teamColours.b, 'B'+i);
      playerGroupB.push(p);
    }
    // GK right
    const gkB = createPlayer(scene, rightX + 80, h/2, teamColours.b, 'B_GK', true);
    playerGroupB.push(gkB);

    // Player main = first of A
    playerMain = playerGroupA[0];
    playerMain.isUser = true; // mark controllable

    // Colliders: players <-> ball bounce
    const allPlayers = [...playerGroupA, ...playerGroupB];
    allPlayers.forEach(p => {
      this.physics.add.collider(p, ball, (pl, b) => {
        // if user pushes into ball and presses shoot, give stronger kick
        if (pl.isUser && pl.wantShoot) {
          shootBallTowardsGoal(scene, pl);
          pl.wantShoot = false;
        } else {
          // small nudged push
          const dx = b.x - pl.x, dy = b.y - pl.y;
          const len = Math.hypot(dx,dy) || 1;
          b.body.velocity.x = (dx/len) * 120 + (b.body.velocity.x*0.6);
          b.body.velocity.y = (dy/len) * 120 + (b.body.velocity.y*0.6);
        }
      });
    });

    // World bounds inside field margins (so ball can't go past visual border)
    ball.body.onWorldBounds = true;

    // Invisible goal triggers
    const leftGoal = this.add.zone(24, h/2, 12, goalH).setOrigin(0,0.5);
    this.physics.add.existing(leftGoal);
    leftGoal.body.setAllowGravity(false);
    leftGoal.body.setImmovable(true);

    const rightGoal = this.add.zone(w-24-12, h/2, 12, goalH).setOrigin(0,0.5);
    this.physics.add.existing(rightGoal);
    rightGoal.body.setAllowGravity(false);
    rightGoal.body.setImmovable(true);

    this.physics.add.overlap(ball, leftGoal, ()=> { goalScored('B'); });
    this.physics.add.overlap(ball, rightGoal, ()=> { goalScored('A'); });

    // Simple AI behaviour for team B (chase ball)
    this.time.addEvent({ loop:true, delay:120, callback:()=>{
      aiUpdate(scene);
    }});

    // HUD
    hudEl = document.getElementById('score');
    updateHud();

    goalTextEl = document.getElementById('goalText');

    // Timer
    timerEvent = this.time.addEvent({ delay:1000, loop:true, callback:()=>{
      matchTime--;
      document.getElementById('time').innerText = matchTime + 's';
      if (matchTime <= 0) {
        timerEvent.remove(false);
        endMatch(scene);
      }
    }});

    // Input for desktop as fallback
    cursors = this.input.keyboard.createCursorKeys();
    this.input.keyboard.on('keydown-SPACE', ()=> {
      playerMain.wantShoot = true;
    });

    // Touch UI display
    if (isMobile) {
      document.getElementById('joyOuter').style.display = 'flex';
      document.getElementById('touchBtns').style.display = 'flex';
      setupJoystick();
      document.getElementById('shootBtn').addEventListener('touchstart', ()=> {
        playerMain.wantShoot = true;
        // small visual feedback
        document.getElementById('shootBtn').style.transform = 'translateY(2px)';
        setTimeout(()=> document.getElementById('shootBtn').style.transform = 'none', 100);
      }, {passive:true});
    } else {
      // show joystick anyway for desktop testing
      document.getElementById('joyOuter').style.display = 'flex';
      document.getElementById('touchBtns').style.display = 'flex';
    }

    // Play button overlay
    document.getElementById('playBtn').addEventListener('click', ()=> {
      document.getElementById('overlay').style.display = 'none';
      // play crowd if available
      const s = scene.sound;
      if (s) {
        try { s.play('crowd', {loop:true, volume:0.2}); } catch(e){}
      }
    });

    document.getElementById('howBtn').addEventListener('click', ()=> {
      alert('Điều khiển trên điện thoại: kéo cần ảo để di chuyển, bấm SÚT để sút. Trên máy tính: dùng phím mũi tên để di chuyển + SPACE để sút.');
    });

    // Give ball a gentle nudge
    ball.body.velocity.x = 80 * (Math.random() > 0.5 ? 1 : -1);
    ball.body.velocity.y = Phaser.Math.Between(-40,40);

    // make players collide with world bounds
    allPlayers.forEach(p=>p.body.setCollideWorldBounds(true));
  }

  function update(time, delta) {
    // handle playerMain movement from joystick or keyboard
    const speed = 160;
    let vx = 0, vy = 0;
    // keyboard
    if (!isMobile) {
      if (cursors.left.isDown) vx = -1;
      if (cursors.right.isDown) vx = 1;
      if (cursors.up.isDown) vy = -1;
      if (cursors.down.isDown) vy = 1;
    }
    // joystick override
    if (joystick.active) {
      const norm = Math.hypot(joystick.dx, joystick.dy);
      if (norm > 6) {
        const nx = joystick.dx / norm;
        const ny = joystick.dy / norm;
        vx = nx; vy = ny;
      }
    }

    playerMain.body.setVelocity(vx * speed, vy * speed);

    // Basic anti-stuck: slow ball slightly
    if (ball.body.velocity.length() > 4) {
      ball.body.velocity.x *= 0.998;
      ball.body.velocity.y *= 0.998;
    }
    // players small drift dampening
    [...playerGroupA, ...playerGroupB].forEach(p => {
      p.body.velocity.x *= 0.96;
      p.body.velocity.y *= 0.96;
    });
  }

  // --- Helpers ---
  function createPlayer(scene, x, y, color, name, isGK=false) {
    const size = isGK ? 22 : 16;
    const shape = scene.add.circle(x, y, size, color);
    scene.physics.add.existing(shape);
    shape.body.setCircle(size);
    shape.body.setBounce(0.8);
    shape.body.setDrag(200, 200);
    shape.name = name;
    shape.isGoalkeeper = !!isGK;
    shape.isUser = false;
    shape.wantShoot = false;
    // show small label
    const lbl = scene.add.text(x, y - size - 8, name.replace('_',''), { font:'12px Arial', fill:'#fff' }).setOrigin(0.5);
    shape._label = lbl;
    // update label position each frame
    scene.events.on('postupdate', ()=> {
      lbl.x = shape.x; lbl.y = shape.y - size - 8;
    });
    return shape;
  }

  function aiUpdate(scene) {
    // simple: each AI (non-GK) move toward ball; GK tries to stay near goal
    playerGroupB.forEach(p => {
      if (p.isGoalkeeper) {
        // move along Y to match ball
        const targetY = Phaser.Math.Clamp(ball.y, 60, scene.scale.height - 60);
        p.body.velocity.y = (targetY - p.y) * 1.2;
        p.body.velocity.x = 0;
      } else {
        // approach ball if far, else small random movement
        const dx = ball.x - p.x, dy = ball.y - p.y;
        const dist = Math.hypot(dx,dy);
        if (dist > 30) {
          p.body.velocity.x = (dx / dist) * 120;
          p.body.velocity.y = (dy / dist) * 120;
        } else {
          p.body.velocity.x = Phaser.Math.Between(-30,30);
          p.body.velocity.y = Phaser.Math.Between(-30,30);
        }
      }
    });

    // Simple teammates A (non-user) basic support movement
    playerGroupA.forEach(p => {
      if (!p.isUser && !p.isGoalkeeper) {
        // move to a position relative to user: simple formation
        const idx = parseInt(p.name.replace('A','')) || 1;
        const baseX = playerMain.x - 40 + (idx * 40);
        const baseY = playerMain.y + (idx-1) * 40;
        const dx = baseX - p.x, dy = baseY - p.y;
        p.body.velocity.x = dx * 0.12;
        p.body.velocity.y = dy * 0.12;
      }
      if (p.isGoalkeeper) {
        // GK left stays near left goal
        const target = { x: scene.scale.width*0.22 - 80, y: scene.scale.height/2 };
        p.body.velocity.x = (target.x - p.x) * 0.03;
        p.body.velocity.y = (target.y - p.y) * 0.03;
      }
    });
  }

  function shootBallTowardsGoal(scene, pl) {
    // determine target goal (right side)
    const goalX = scene.scale.width - 24;
    const goalY = scene.scale.height/2;
    const dx = goalX - pl.x, dy = goalY - pl.y;
    const len = Math.hypot(dx,dy) || 1;
    const power = 520 + Phaser.Math.Between(-60,60);
    ball.body.velocity.x = (dx/len) * power;
    ball.body.velocity.y = (dy/len) * power * 0.6;
    // play sound
    try { scene.sound.play('kick', { volume:0.4 }); } catch(e){}
  }

  function goalScored(side) {
    // side: 'A' means team A scored? (we set earlier calls: if ball enters rightGoal => A)
    if (side === 'A') scoreA++;
    if (side === 'B') scoreB++;
    updateHud();
    showGoalEffect();
    // reset ball & players positions shortly
    game.scene.scenes[0].time.delayedCall(900, () => {
      resetPositions(game.scene.scenes[0]);
    });
  }

  function showGoalEffect() {
    goalTextEl.style.display = 'block';
    goalTextEl.style.transform = 'scale(0.7)';
    goalTextEl.style.opacity = '1';
    // animate with JS
    let t = 0;
    const iv = setInterval(()=> {
      t += 0.06;
      const s = 1 + Math.sin(t*6) * 0.12 + 0.2;
      goalTextEl.style.transform = 'translateX(-50%) scale(' + s + ')';
      if (t > 1.2) {
        clearInterval(iv);
        goalTextEl.style.display = 'none';
      }
    }, 16);
    // play crowd
    try { game.scene.scenes[0].sound.play('crowd', { volume:0.6 }); } catch(e){}
  }

  function resetPositions(scene) {
    // ball center
    ball.x = scene.scale.width/2;
    ball.y = scene.scale.height/2;
    ball.body.setVelocity(0,0);
    // players back to spawn rough positions
    const w = scene.scale.width, h = scene.scale.height;
    const leftX = w*0.22, rightX = w*0.78;
    const rows = [-80, 0, 80];
    for (let i=0;i<3;i++){
      playerGroupA[i].x = leftX; playerGroupA[i].y = h/2 + rows[i];
      playerGroupB[i].x = rightX; playerGroupB[i].y = h/2 + rows[i];
    }
    playerGroupA[3].x = leftX - 80; playerGroupA[3].y = h/2; // GK
    playerGroupB[3].x = rightX + 80; playerGroupB[3].y = h/2; // GK
  }

  function endMatch(scene) {
    // show overlay with result
    const ov = document.getElementById('overlay');
    ov.style.display = 'flex';
    ov.querySelector('.green').innerText = 'Chơi lại';
    ov.querySelector('.green').onclick = ()=> {
      // reset and restart
      scoreA = 0; scoreB = 0; matchTime = 120;
      document.getElementById('time').innerText = matchTime + 's';
      updateHud();
      resetPositions(scene);
      ov.style.display = 'none';
      // restart timer
      timerEvent = scene.time.addEvent({ delay:1000, loop:true, callback:()=>{
        matchTime--;
        document.getElementById('time').innerText = matchTime + 's';
        if (matchTime <= 0) { timerEvent.remove(false); endMatch(scene); }
      }});
    };
    ov.querySelector('#howBtn').style.display = 'none';
    ov.querySelector('#playBtn').style.display = 'none';
    ov.querySelector('.small').innerText = `Kết quả: VN ${scoreA} - ${scoreB} TH`;
  }

  function updateHud() {
    document.getElementById('score').innerText = `VN ${scoreA} - ${scoreB} TH`;
  }

  // Joystick implementation (simple)
  function setupJoystick() {
    const outer = document.getElementById('joyOuter');
    const stick = document.getElementById('stick');
    const outerRect = outer.getBoundingClientRect();
    const centerX = outerRect.left + outerRect.width/2;
    const centerY = outerRect.top + outerRect.height/2;
    // pointer events
    let pointerId = null;
    stick.addEventListener('touchstart', (e) => {
      e.preventDefault();
      pointerId = e.changedTouches[0].identifier;
      joystick.active = true;
      joystick.startX = e.changedTouches[0].clientX;
      joystick.startY = e.changedTouches[0].clientY;
    }, {passive:false});
    stick.addEventListener('touchmove', (e) => {
      e.preventDefault();
      for (let t of e.changedTouches) {
        if (t.identifier === pointerId) {
          const dx = t.clientX - (centerX);
          const dy = t.clientY - (centerY);
          const max = 36;
          const dist = Math.hypot(dx,dy);
          const clamped = dist > max ? max / dist : 1;
          const px = dx * clamped;
          const py = dy * clamped;
          stick.style.transform = `translate(${px}px, ${py}px)`;
          joystick.dx = px; joystick.dy = py;
        }
      }
    }, {passive:false});
    stick.addEventListener('touchend', (e) => {
      e.preventDefault();
      joystick.active = false; joystick.dx = 0; joystick.dy = 0;
      stick.style.transform = '';
      pointerId = null;
    }, {passive:false});

    // Also allow touching outer area to start
    outer.addEventListener('touchstart', (e)=> {
      e.preventDefault();
      const t = e.changedTouches[0];
      pointerId = t.identifier;
      joystick.active = true;
      const dx = t.clientX - centerX;
      const dy = t.clientY - centerY;
      joystick.dx = dx; joystick.dy = dy;
      stick.style.transform = `translate(${dx}px, ${dy}px)`;
    }, {passive:false});
    outer.addEventListener('touchmove', (e)=> {
      e.preventDefault();
      for (let t of e.changedTouches) {
        if (t.identifier === pointerId) {
          const dx = t.clientX - centerX;
          const dy = t.clientY - centerY;
          const max = 36;
          const dist = Math.hypot(dx,dy);
          const clamped = dist > max ? max / dist : 1;
          const px = dx * clamped;
          const py = dy * clamped;
          stick.style.transform = `translate(${px}px, ${py}px)`;
          joystick.dx = px; joystick.dy = py;
        }
      }
    }, {passive:false});
    outer.addEventListener('touchend', (e)=> {
      e.preventDefault();
      joystick.active = false; joystick.dx = 0; joystick.dy = 0;
      stick.style.transform = '';
      pointerId = null;
    }, {passive:false});
  }

  // expose restart if wanted
  window.resetDemo = function() {
    scoreA = 0; scoreB = 0; matchTime = 120; updateHud();
    resetPositions(game.scene.scenes[0]);
  };

  </script>
</body>
</html>
