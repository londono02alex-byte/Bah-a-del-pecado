agregale el mapa de Cartagena al juego 
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Bahía del Pecado</title>
  <style>
    body {
      margin: 0;
      background-color: #000;
      color: #fff;
      font-family: 'Poppins', sans-serif;
      text-align: center;
      overflow-x: hidden;
    }
    header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 20px 40px;
      background: rgba(0,0,0,0.8);
      position: fixed;
      width: 100%;
      top: 0;
      z-index: 10;
    }
    header h1 {
      font-family: 'Times New Roman', serif;
      font-size: 26px;
      color: white;
    }
    nav a {
      color: #ccc;
      margin: 0 15px;
      text-decoration: none;
      transition: 0.3s;
    }
    nav a:hover {
      color: #ffb400;
    }
    section {
      padding: 100px 20px;
      max-width: 900px;
      margin: auto;
    }
    h2 {
      font-size: 2.5em;
      color: #ffb400;
      margin-bottom: 10px;
    }
    .btn {
      background-color: #ffb400;
      color: #000;
      padding: 12px 30px;
      border-radius: 10px;
      font-weight: bold;
      text-decoration: none;
      transition: 0.3s;
    }
    .btn:hover {
      background-color: #fff;
    }
    canvas {
      width: 100%;
      height: 400px;
      background: #111;
      border-radius: 15px;
      margin-top: 40px;
    }
  </style>
</head>

<body>

  <header>
    <h1>Bahía del Pecado</h1>
    <nav>
      <a href="#mundo">El Mundo</a>
      <a href="#trailer">Tráiler</a>
      <a href="#juego">El Juego</a>
      <a href="#arte">Arte</a>
      <a href="#noticias">Noticias</a>
    </nav>
  </header>

  <section id="inicio">
    <h2>BAHÍA DEL PECADO</h2>
    <p><em>"El que manda no se anuncia, se siente."</em></p>
    <p>Sumérgete en un mundo abierto hiperrealista en 4K.  
    Eres <strong>Alex Vega</strong>, un renegado que busca venganza y control en las calles corruptas de Cartagena.</p>
    <a class="btn" href="#juego">Explora la Bahía</a>
  </section>

  <section id="mundo">
    <h2>🌆 El Mundo</h2>
    <p>Cartagena, 2042. Una ciudad dominada por el lujo, el crimen y los secretos.  
    Desde los barrios antiguos hasta las zonas tecnológicas del puerto, cada esquina es un riesgo… y una oportunidad.  
    La Bahía vibra con luces de neón, música urbana y un aire de peligro constante.</p>
  </section>

  <section id="trailer">
    <h2>🎬 Tráiler Oficial</h2>
    <p>Disfruta el avance cinematográfico con ritmo urbano y estética de reguetón costeño.</p>
    <video controls width="80%">
      <source src="trailer-bahia.mp4" type="video/mp4">
      Tu navegador no soporta video.
    </video>
  </section>

  <section id="juego">
    <h2>🎮 El Juego</h2>
    <p>Explora libremente un mundo 3D con total libertad.  
    Controla a Alex Vega con movimiento fluido, cámara libre y mecánicas urbanas interactivas.  
    Puedes correr, saltar, explorar y desbloquear misiones ocultas en la Bahía.</p>

    <canvas id="gameCanvas"></canvas>
  </section>

  <section id="arte">
    <h2>🎨 Arte y Estilo</h2>
    <p>Gráficos realistas con tonos negros y dorados, animaciones suaves y reflejos en 4K.  
    Cada textura fue creada para resaltar el calor, el ritmo y la intensidad de la vida caribeña moderna.</p>
  </section>

  <section id="noticias">
    <h2>📰 Noticias</h2>
    <p>Actualizaciones semanales, nuevos personajes, misiones y eventos exclusivos.  
    Mantente atento a las colaboraciones con artistas urbanos y DJ del Caribe.</p>
  </section>

  <footer>
    <p>© 2025 Bahía del Pecado | Creado por Alex Vega y equipo IA</p>
  </footer>

  <script>
    // Mini motor de prueba del personaje "Alex" en canvas
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");
    let x = 50, y = 200;
    const speed = 5;

    function draw() {
      ctx.fillStyle = "#111";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "#ffb400";
      ctx.fillRect(x, y, 40, 40);
      ctx.fillText("Alex Vega", x, y - 10);
    }

    function update() {
      draw();
      requestAnimationFrame(update);
    }

    document.addEventListener("keydown", e => {
      if (e.key === "ArrowRight") x += speed;
      if (e.key === "ArrowLeft") x -= speed;
      if (e.key === "ArrowUp") y -= speed;
      if (e.key === "ArrowDown") y += speed;
    });

    update();
  </script>

</body>
</html>