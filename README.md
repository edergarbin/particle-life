# Simulação de Partículas

Este é um projeto simples de simulação de partículas usando JavaScript e a biblioteca `canvas-sketch`. A ideia é fornecer um exemplo de como criar uma simulação interativa onde podemos ajustar parâmetros como número de partículas, velocidade, coesão, separação e alinhamento.

Eu acho fascinante ver como podemos simular comportamentos que identificamos na natureza usando código. Este projeto é uma pequena demonstração de como conceitos como coesão, separação e alinhamento, observados em cardumes de peixes e bandos de pássaros, podem ser recriados digitalmente. Foi apenas um projeto de estudos, mas espero que você goste ou ache interessante.

## Funcionalidades

- Simulação de partículas com comportamento de boids (cohesion, separation, alignment).
- Controles interativos para ajustar os parâmetros da simulação.
- Renderização dinâmica usando `canvas-sketch`.

## Instalação

1. Clone o repositório:
    ```bash
    git clone https://github.com/edergarbin/particle-life.git
    cd particle-life
    ```

2. Instale as dependências:
    ```bash
    npm install
    ```

3. Execute o projeto:
    ```bash
    canvas-sketch index.js --open
    ```

## Uso

- Após iniciar o projeto, uma janela do navegador será aberta com a simulação.
- Você pode ajustar os parâmetros da simulação usando os controles na parte superior esquerda da tela.

## Como Funciona

Aqui está uma breve descrição do código:

#### index.js

```javascript
import canvasSketch from 'canvas-sketch';
import random from 'canvas-sketch-util/random';

// Configurações do sketch
const settings = {
  dimensions: [800, 800],
  animate: true
};

// Parâmetros ajustáveis
const params = {
  particleCount: 100,
  maxSpeed: 2,
  cohesion: 0.01,
  separation: 0.05,
  alignment: 0.05
};

// Adiciona controles HTML
const addControls = () => {
  const controls = document.createElement('div');
  controls.innerHTML = `
    <div style="position: absolute; top: 10px; left: 10px; background: rgba(255, 255, 255, 0.8); padding: 10px; border-radius: 8px;">
      <label>Particle Count: <input type="number" id="particleCount" value="${params.particleCount}" min="10" max="200"></label><br>
      <label>Max Speed: <input type="range" id="maxSpeed" value="${params.maxSpeed}" min="0.1" max="10" step="0.1"></label><br>
      <label>Cohesion: <input type="range" id="cohesion" value="${params.cohesion}" min="0" max="0.1" step="0.01"></label><br>
      <label>Separation: <input type="range" id="separation" value="${params.separation}" min="0" max="0.1" step="0.01"></label><br>
      <label>Alignment: <input type="range" id="alignment" value="${params.alignment}" min="0" max="0.1" step="0.01"></label>
    </div>
  `;
  document.body.appendChild(controls);

  // Atualiza os parâmetros com base nos controles
  document.getElementById('particleCount').addEventListener('input', (event) => {
    params.particleCount = parseInt(event.target.value, 10);
    resetParticles();
  });

  document.getElementById('maxSpeed').addEventListener('input', (event) => {
    params.maxSpeed = parseFloat(event.target.value);
  });

  document.getElementById('cohesion').addEventListener('input', (event) => {
    params.cohesion = parseFloat(event.target.value);
  });

  document.getElementById('separation').addEventListener('input', (event) => {
    params.separation = parseFloat(event.target.value);
  });

  document.getElementById('alignment').addEventListener('input', (event) => {
    params.alignment = parseFloat(event.target.value);
  });
};

// Classe que representa um vetor 2D
class Vector2 {
  constructor(x = 0, y = 0) {
    this.x = x;
    this.y = y;
  }

  // Métodos para operações vetoriais
  add(v) {
    this.x += v.x;
    this.y += v.y;
    return this;
  }

  subtract(v) {
    this.x -= v.x;
    this.y -= v.y;
    return this;
  }

  multiply(n) {
    this.x *= n;
    this.y *= n;
    return this;
  }

  divide(n) {
    this.x /= n;
    this.y /= n;
    return this;
  }

  distance(v) {
    let dx = this.x - v.x;
    let dy = this.y - v.y;
    return Math.sqrt(dx * dx + dy * dy);
  }

  limit(max) {
    let magnitude = Math.sqrt(this.x * this.x + this.y * this.y);
    if (magnitude > max) {
      this.x = (this.x / magnitude) * max;
      this.y = (this.y / magnitude) * max;
    }
    return this;
  }

  normalize() {
    let magnitude = Math.sqrt(this.x * this.x + this.y * this.y);
    if (magnitude > 0) {
      this.x /= magnitude;
      this.y /= magnitude;
    }
    return this;
  }

  clone() {
    return new Vector2(this.x, this.y);
  }
}

// Classe Partícula que representa cada partícula na simulação
class Particle {
  constructor(x, y) {
    this.position = new Vector2(x, y);
    this.velocity = new Vector2(random.range(-1, 1), random.range(-1, 1));
    this.acceleration = new Vector2();
  }

  // Atualiza a posição da partícula
  update(particles) {
    this.flock(particles);
    this.velocity.add(this.acceleration);
    this.velocity.limit(params.maxSpeed);
    this.position.add(this.velocity);
    this.acceleration.multiply(0);
  }

  applyForce(force) {
    this.acceleration.add(force);
  }

  // Aplica as regras de coesão, separação e alinhamento
  flock(particles) {
    let alignment = new Vector2();
    let cohesion = new Vector2();
    let separation = new Vector2();
    let count = 0;

    particles.forEach(particle => {
      let d = this.position.distance(particle.position);
      if (d > 0 && d < 50) {
        alignment.add(particle.velocity);
        cohesion.add(particle.position);
        let diff = this.position.clone().subtract(particle.position);
        diff.divide(d);
        separation.add(diff);
        count++;
      }
    });

    if (count > 0) {
      alignment.divide(count).normalize().multiply(params.alignment);
      cohesion.divide(count).subtract(this.position).normalize().multiply(params.cohesion);
      separation.divide(count).normalize().multiply(params.separation);
    }

    this.applyForce(alignment);
    this.applyForce(cohesion);
    this.applyForce(separation);
  }

  // Garante que a partícula reapareça do outro lado se sair dos limites
  edges() {
    if (this.position.x > settings.dimensions[0]) this.position.x = 0;
    if (this.position.x < 0) this.position.x = settings.dimensions[0];
    if (this.position.y > settings.dimensions[1]) this.position.y = 0;
    if (this.position.y < 0) this.position.y = settings.dimensions[1];
  }

  // Desenha a partícula no canvas
  draw(context) {
    context.fillStyle = 'white';
    context.beginPath();
    context.arc(this.position.x, this.position.y, 2, 0, Math.PI * 2);
    context.fill();
  }
}

let particles = [];

// Função para reiniciar as partículas com base no novo número de partículas
const resetParticles = () => {
  particles = [];
  for (let i = 0; i < params.particleCount; i++) {
    particles.push(new Particle(random.range(0, settings.dimensions[0]), random.range(0, settings.dimensions[1])));
  }
};

// Inicialização do sketch
canvasSketch(() => {
  // Adiciona os controles HTML após o conteúdo do DOM estar carregado
  window.addEventListener('DOMContentLoaded', (event) => {
    addControls();
    resetParticles();
  });

  return {
    render({ context, width, height }) {
      // Limpa o canvas
      context.fillStyle = 'black';
      context.fillRect(0, 0, width, height);

      // Atualiza e desenha cada partícula
      particles.forEach(particle => {
        particle.edges();
        particle.update(particles);
        particle.draw(context);
      });
    },
    resize({ width, height }) {
      // Reposiciona as partículas ao redimensionar o canvas
      particles.forEach(particle => {
        particle.position.x = random.range(0, width);
        particle.position.y = random.range(0, height);
      });
    }
  };
}, settings);
