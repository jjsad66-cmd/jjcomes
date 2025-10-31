<!doctype html>
<html lang="jh">
<head>
<meta charset="utf-8" />
<title>Warrior and The Evil Presence</title>
<style>
  html,body{height:100%;margin:0;background:#0b0b12;color:#ddd;font-family:system-ui,Segoe UI,Roboto,Arial;}
  #game {display:block;margin:18px auto;border:6px solid #1f2937;background:#06060a; image-rendering: pixelated;}
  .ui {width:800px;margin:10px auto;text-align:center;}
  .hint {font-size:14px;color:#9aa5b1;}
</style>
</head>
<body>
  <div class="ui">
    <h1>Warrior vs The Evil Presence</h1>
    <div class="hint">Move: WASD / Arrows · Attack: Space · Restart: R</div>
  </div>
  <canvas id="game" width="800" height="500"></canvas>
  <script>
  // -----------------------------
  // Simple 2D Game (single file)
  // -----------------------------
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');

  const W = canvas.width, H = canvas.height;
  let keys = {};

  addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
  addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

  // Utilities
  function clamp(v, a, b){ return Math.max(a, Math.min(b, v)); }
  function dist(a,b){ return Math.hypot(a.x-b.x, a.y-b.y); }
  function rectColl(a,b){ return !(a.x+a.w < b.x || a.x > b.x+b.w || a.y+a.h < b.y || a.y > b.y+b.h); }

  // Game Entities
  class Warrior {
    constructor(){
      this.w = 32; this.h = 40;
      this.x = 80; this.y = H/2 - this.h/2;
      this.speed = 220; // px/sec
      this.maxHp = 100; this.hp = this.maxHp;
      this.attackCooldown = 0; // seconds
      this.attackRate = 0.5;
      this.attackRange = 48;
      this.attackPower = 20;
      this.isAttacking = false;
      this.invulnerable = 0;
    }
    update(dt){
      let vx = 0, vy = 0;
      if (keys['arrowleft']||keys['a']) vx = -1;
      if (keys['arrowright']||keys['d']) vx = 1;
      if (keys['arrowup']||keys['w']) vy = -1;
      if (keys['arrowdown']||keys['s']) vy = 1;
      const mag = Math.hypot(vx,vy) || 1;
      this.x += vx/mag * this.speed * dt;
      this.y += vy/mag * this.speed * dt;
      this.x = clamp(this.x, 0, W - this.w);
      this.y = clamp(this.y, 0, H - this.h);

      if (this.attackCooldown > 0) this.attackCooldown -= dt;
      if (this.invulnerable > 0) this.invulnerable -= dt;

      if ((keys[' '] || keys['space']) && this.attackCooldown <= 0) {
        this.attackCooldown = this.attackRate;
        this.isAttacking = 0.12; // seconds of attack animation
      }
      if (this.isAttacking) this.isAttacking -= dt;
      else this.isAttacking = Math.max(0, this.isAttacking);

      if (this.hp <= 0) gameOver = true;
    }
    attackBox(){
      // returns attack rect
      const cx = this.x + this.w/2;
      const cy = this.y + this.h/2;
      return { x: cx, y: cy - 18, w: this.attackRange, h: 36 };
    }
    draw(){
      // body
      ctx.save();
      // flicker when invulnerable
      if (this.invulnerable > 0 && Math.floor(this.invulnerable*10)%2==0) ctx.globalAlpha = 0.4;
      ctx.fillStyle = '#6db6ff';
      ctx.fillRect(this.x, this.y, this.w, this.h);
      // head
      ctx.fillStyle = '#08304a';
      ctx.fillRect(this.x+6, this.y-8, this.w-12, 14);
      // sword when attacking
      if (this.isAttacking > 0){
        const ab = this.attackBox();
        ctx.fillStyle = '#ffd59a';
        ctx.fillRect(ab.x+this.w/2, ab.y, ab.w/1.5, ab.h);
      }
      ctx.restore();

      // draw HP
      drawBar(12, 12, 220, 16, this.hp / this.maxHp, 'Warrior', '#6db6ff');
    }
  }

  class EvilPresence {
    constructor(){
      this.w = 120; this.h = 120;
      this.x = W - 180; this.y = H/2 - this.h/2;
      this.maxHp = 400; this.hp = this.maxHp;
      this.phase = 1;
      this.timer = 0;
      this.spawnCooldown = 0;
      this.minions = [];
      this.anger = 0;
    }
    update(dt){
      this.timer += dt;
      // simple floating movement
      this.y += Math.sin(this.timer*1.2) * 10 * dt;

      // Anger rises as it loses HP -> more aggressive spawns
      this.anger = 1 - (this.hp / this.maxHp);

      // spawn minions
      this.spawnCooldown -= dt;
      const spawnRate = clamp(2.5 - this.anger*2, 0.5, 2.5); // faster when angry
      if (this.spawnCooldown <= 0){
        this.spawnCooldown = spawnRate;
        this.minions.push(new Minion(this.x + Math.random()*this.w, this.y + this.h/2 + (Math.random()-0.5)*60));
      }

      // update minions
      for (let m of this.minions) m.update(dt);
      this.minions = this.minions.filter(m => !m.dead);

      // change phase visually
      if (this.hp <= this.maxHp * 0.5 && this.phase === 1) {
        this.phase = 2;
      }

      // defeat?
      if (this.hp <= 0) {
        this.hp = 0;
        bossDefeated = true;
      }
    }
    draw(){
      // aura
      const cx = this.x + this.w/2, cy = this.y + this.h/2;
      const pulse = 6 + Math.sin(this.timer*6) * 4;
      ctx.save();
      // aura gradient
      const g = ctx.createRadialGradient(cx, cy, 10, cx, cy, this.w*0.9);
      g.addColorStop(0, `rgba(200,40,120,${0.8 + this.anger*0.4})`);
      g.addColorStop(1, 'rgba(6,10,20,0)');
      ctx.fillStyle = g;
      ctx.beginPath();
      ctx.ellipse(cx, cy, this.w*0.8 + pulse, this.h*0.6 + pulse, 0, 0, Math.PI*2);
      ctx.fill();

      // core body
      ctx.fillStyle = this.phase===1 ? '#7b1b4a' : '#ff3860'; // enraged color change
      roundRect(ctx, this.x, this.y, this.w, this.h, 28, true, false);

      // eyes
      ctx.fillStyle = '#fff';
      ctx.fillRect(this.x+18, this.y+28, 14, 8);
      ctx.fillRect(this.x+88, this.y+28, 14, 8);
      ctx.fillStyle = '#000';
      ctx.fillRect(this.x+22, this.y+30, 6, 4);
      ctx.fillRect(this.x+92, this.y+30, 6, 4);

      // pulsating center
      ctx.fillStyle = 'rgba(255,255,120,0.14)';
      ctx.beginPath();
      ctx.ellipse(cx, cy+10, 28+Math.sin(this.timer*3)*6, 18+Math.cos(this.timer*2)*5, 0, 0, Math.PI*2);
      ctx.fill();

      ctx.restore();

      // minions
      for (let m of this.minions) m.draw();

      // draw boss HP
      drawBar(W - 232, 12, 220, 16, this.hp / this.maxHp, 'Evil Presence', '#ff5c8a');
    }
    hit(amount){
      this.hp -= amount;
      if (this.hp < 0) this.hp = 0;
    }
  }

  class Minion {
    constructor(x,y){
      this.w = 22; this.h = 22;
      this.x = x; this.y = y;
      this.speed = 80 + Math.random()*40;
      this.hp = 12;
      this.dead = false;
      this.vx = -1; // moves left toward player zone
    }
    update(dt){
      this.x += this.vx * this.speed * dt;
      // simple behavior: hover slightly
      this.y += Math.sin(Date.now()/300 + this.x) * 0.4;
      if (this.x < -50) this.dead = true;
      if (this.hp <= 0) this.dead = true;
    }
    draw(){
      ctx.save();
      ctx.fillStyle = '#ffb3c9';
      ctx.fillRect(this.x, this.y, this.w, this.h);
      ctx.restore();
    }
  }

  // Helpers
  function drawBar(x,y,w,h,ratio,label,color){
    ctx.save();
    ctx.fillStyle = '#1b2430';
    ctx.fillRect(x-2, y-2, w+4, h+4);
    ctx.fillStyle = '#222';
    ctx.fillRect(x, y, w, h);
    ctx.fillStyle = color;
    ctx.fillRect(x, y, Math.max(0, w*ratio), h);
    ctx.fillStyle = '#e9eef5';
    ctx.font = '13px system-ui';
    ctx.textAlign = 'center';
    ctx.fillText(label + ' — ' + Math.round(ratio*100) + '%', x + w/2, y + h - 4);
    ctx.restore();
  }
  function roundRect(ctx, x, y, w, h, r, fill, stroke){
    if (r === undefined) r = 5;
    ctx.beginPath();
    ctx.moveTo(x+r, y);
    ctx.arcTo(x+w, y,   x+w, y+h, r);
    ctx.arcTo(x+w, y+h, x,   y+h, r);
    ctx.arcTo(x,   y+h, x,   y,   r);
    ctx.arcTo(x,   y,   x+w, y,   r);
    ctx.closePath();
    if (fill) ctx.fill();
    if (stroke) ctx.stroke();
  }

  // Game state
  let player = new Warrior();
  let boss = new EvilPresence();
  let last = performance.now();
  let gameOver = false;
  let bossDefeated = false;
  let score = 0;
  let playPaused = false;

  function reset(){
    player = new Warrior();
    boss = new EvilPresence();
    last = performance.now();
    gameOver = false;
    bossDefeated = false;
    score = 0;
  }

  // Collisions, combat
  function handleCombat(dt){
    // When player attacks, check boss and minions in attack box (simple rectangular)
    if (player.isAttacking > 0.05) {
      const ab = player.attackBox();
      // check boss: approximate boss rect
      const bossRect = { x: boss.x, y: boss.y, w: boss.w, h: boss.h };
      if (rectColl(ab, bossRect)) {
        // deal damage and brief stun/invuln
        boss.hit(player.attackPower);
        score += 10;
      }
      // minions
      for (let m of boss.minions){
        if (!m.dead && rectColl(ab, {x:m.x,y:m.y,w:m.w,h:m.h})){
          m.hp -= player.attackPower;
          score += 4;
        }
      }
    }

    // minions hitting player
    for (let m of boss.minions){
      if (!m.dead && rectColl({x:player.x,y:player.y,w:player.w,h:player.h}, {x:m.x,y:m.y,w:m.w,h:m.h})){
        if (player.invulnerable <= 0) {
          player.hp -= 8;
          player.invulnerable = 0.9;
        }
      }
    }

    // boss aura damage if too close
    const d = dist({x:player.x + player.w/2, y:player.y + player.h/2}, {x: boss.x + boss.w/2, y: boss.y + boss.h/2});
    if (d < 120 && player.invulnerable <= 0){
      player.hp -= 14 * (1/60); // small continuous damage
      // set short invul so damage isn't instant-death
      player.invulnerable = 0.4;
    }
  }

  // Main loop
  function loop(t){
    const dt = Math.min(0.05, (t - last) / 1000);
    last = t;
    if (!playPaused && !gameOver && !bossDefeated){
      player.update(dt);
      boss.update(dt);
      handleCombat(dt);
    }

    // Draw
    ctx.clearRect(0,0,W,H);
    // background
    ctx.fillStyle = '#05050a';
    ctx.fillRect(0,0,W,H);

    // scenery: ruins
    for (let i=0;i<6;i++){
      ctx.fillStyle = 'rgba(80,80,90,0.04)';
      ctx.fillRect(40 + i*120, H - 80 - (i%2)*8, 60, 80);
    }

    boss.draw();
    player.draw();

    // HUD & messages
    ctx.save();
    ctx.fillStyle = '#9fb0c8';
    ctx.font = '14px system-ui';
    ctx.fillText('Score: ' + score, W/2, 20);
    if (gameOver){
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(W/2 -220, H/2 -70, 440, 140);
      ctx.fillStyle = '#ff6b6b';
      ctx.font = '30px system-ui';
      ctx.textAlign = 'center';
      ctx.fillText('You have fallen...', W/2, H/2 - 8);
      ctx.fillStyle = '#dfe9f5';
      ctx.font = '16px system-ui';
      ctx.fillText('Press R to restart', W/2, H/2 + 26);
    } else if (bossDefeated){
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(W/2 -260, H/2 -100, 520, 200);
      ctx.fillStyle = '#9dffb8';
      ctx.font = '36px system-ui';
      ctx.textAlign = 'center';
      ctx.fillText('Victory! The Evil Presence is vanquished.', W/2, H/2 - 8);
      ctx.fillStyle = '#e8f6ff';
      ctx.font = '16px system-ui';
      ctx.fillText('You fought bravely. Press R to challenge it again.', W/2, H/2 + 36);
    }
    ctx.restore();

    requestAnimationFrame(loop);
  }

  // Restart key
  addEventListener('keydown', e => {
    if (e.key.toLowerCase() === 'r') reset();
  });

  // Start
  requestAnimationFrame(loop);

  // Small tips for customizing the game (left as comments):
  // - Change boss.maxHp to adjust difficulty
  // - Change player.attackPower or attackRate for faster/slower combat
  // - Add more boss abilities in EvilPresence.update() for variety
  // - Replace simple rectangles with image sprites for polish

  </script>
</body>
</html>
