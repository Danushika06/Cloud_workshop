# Cloud Workshop — Containerised Typing Game

**A neon falling-words typing game built with React and Vite, used as the demo app for a Docker and AWS (EC2 and ECR) containerisation workshop.**

![React](https://img.shields.io/badge/React-61DAFB?style=flat&logo=react&logoColor=black) ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) ![Vite](https://img.shields.io/badge/Vite-646CFF?style=flat&logo=vite&logoColor=white) ![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?style=flat&logo=tailwindcss&logoColor=white) ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white) ![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat&logo=nginx&logoColor=white) ![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonwebservices&logoColor=white)

## Overview

This repo is a small workshop exercise on taking a web app from `npm run dev` on a laptop to a container running on a cloud VM. The app itself is a one-screen typing game, "Workshop Game": words fall down the screen and you type them before they reach the bottom. The point of the repo is the packaging around it: a multi-stage `Dockerfile`, an `nginx.conf`, and the steps I followed for pushing the image to Amazon ECR and running it on EC2.

Status: a finished workshop demo, not a product. The code, workshop steps and architecture diagram come from a workshop (KCT); the original README in this repo links to `duvarakesh1907/KCT-workshop` as the workshop repository, so this is my copy of that material rather than something I designed from scratch. The game has no backend, no saved scores and no tests.

## Key Features

- **Falling-word typing game** — `src/components/NeonTypeGame.tsx` spawns words from `WORD_LIST` (about 150 words in `src/data/words.ts`) at positions between 10 and 90 percent of the screen width, and moves them down on every animation frame.
- **Targeting and scoring** — typing the first letter locks onto the lowest matching word; a wrong key on a locked word flags an error. Completing a word scores `length x 10 x difficulty`.
- **Lives and difficulty** — 5 lives, one lost per word that reaches the bottom; difficulty rises by 0.05 per completed word, capped at 5, and shortens the spawn interval (base 1500 ms).
- **Live WPM** — computed from characters typed (5 characters per word) over elapsed time.
- **Synthesised sound** — `src/utils/SoundManager.ts` generates keystroke, explosion, error and start sounds with the Web Audio API, so there are no audio files.
- **Particle explosions and neon styling** — custom CSS theme tokens in `src/index.css` and the Orbitron / JetBrains Mono fonts.
- **Production container** — multi-stage Docker build: Node 20 builds the app, nginx:alpine serves `dist/`, with a `/health` endpoint and a container `HEALTHCHECK`.

## How It Works

`src/main.tsx` mounts `App` (`src/app.tsx`), which renders only `NeonTypeGame`. The game keeps its fast-changing state (word list, score, lives, difficulty) in refs and copies it into React state for rendering. A `requestAnimationFrame` loop spawns, moves and removes words, and a `keydown` listener handles input (alphanumeric keys only; Enter starts or restarts). The game has three states: `start`, `playing` and `gameover`.

Deployment flow, as set out in the workshop steps:

1. `docker build -t workshop-game .` runs `npm install` and `npm run build` in a `node:20-slim` stage (the Dockerfile notes Alpine causes problems with the `@tailwindcss/oxide` native bindings).
2. The second stage copies `dist/` into `nginx:alpine` with `nginx.conf`: gzip, security headers, one-year immutable caching for `/assets/`, a fallback to `index.html` for all routes, and `GET /health` returning `OK`.
3. The image is tagged and pushed to a private ECR repository created with `aws ecr create-repository`, then pulled and run on an Amazon Linux EC2 instance with `docker run -d -p 8080:80`.

The repo includes an architecture diagram (`KCT-workshop.drawio.png`) described as a multi-AZ deployment in the Mumbai region with a VPC, public and private subnets, ALB, NAT gateway, ECR and app servers. That diagram shows the target design from the workshop; the repo contains no infrastructure code for it.

## Tech Stack

- **Frontend:** React 19, TypeScript 5.8, Vite 6, Tailwind CSS 4 (`@tailwindcss/vite`), lucide-react icons, Framer Motion for animation (imported as `framer-motion`)
- **Audio:** Web Audio API
- **Serving:** nginx (alpine) in Docker
- **Cloud (workshop steps):** AWS EC2, Amazon ECR, AWS CLI

## Getting Started

### Prerequisites
- Node.js 20 or newer
- Docker, for the container route
- For the cloud steps: an AWS account, AWS CLI configured and an EC2 instance on Amazon Linux

### Run locally
```bash
git clone https://github.com/Danushika06/Cloud_workshop.git
cd Cloud_workshop
npm install
npm run dev
```
The dev server runs on http://localhost:3000 (`vite --port=3000 --host=0.0.0.0`).

### Run in Docker
```bash
docker build -t workshop-game .
docker run -d -p 8080:80 --name workshop-game workshop-game
```
Open http://localhost:8080. Stop it with `docker stop workshop-game` and `docker rm workshop-game`.

### Push to ECR and run on EC2
On an Amazon Linux EC2 instance with Git and Docker installed, from the cloned repo (replace `<account-id>` and the region; the workshop used `ap-south-1`):
```bash
aws ecr create-repository --repository-name workshop-game --region ap-south-1
docker build -t workshop-game .
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com
docker tag workshop-game:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/workshop-game:latest
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/workshop-game:latest
docker run -d -p 8080:80 --name workshop-game <account-id>.dkr.ecr.ap-south-1.amazonaws.com/workshop-game:latest
```
Open `http://<EC2-PUBLIC-IP>:8080` and allow inbound port 8080 in the instance's security group.

### Known build issue
`NeonTypeGame.tsx` imports from `framer-motion`, but `package.json` lists the `motion` package and not `framer-motion`. If `npm run build` fails with an unresolved import, install `framer-motion` or change the import to `motion/react`. I have not run the build to confirm either way.

## Usage

Press Enter (or click the start button), type the first letter of a falling word to target it, finish the word before it reaches the red zone, and keep at least one of 5 lives. `npm run lint` runs `tsc --noEmit`.

## Project Structure

```
Cloud_workshop/
├── src/
│   ├── main.tsx                      # React entry point
│   ├── app.tsx                       # renders the game
│   ├── components/NeonTypeGame.tsx   # game loop, input, scoring, UI
│   ├── data/words.ts                 # word list
│   ├── utils/SoundManager.ts         # Web Audio sound effects
│   └── index.css                     # Tailwind theme and effects
├── Dockerfile                        # multi-stage: node build, nginx serve
├── nginx.conf                        # gzip, headers, SPA fallback, /health
├── .dockerignore
├── vite.config.ts                    # React and Tailwind plugins
├── KCT-workshop.drawio.png           # workshop AWS architecture diagram
└── README.md
```

---

## Contact

<table>
  <tr><td><b>GitHub:</b> <a href="https://github.com/Danushika06">Danushika06</a></td><td><b>LinkedIn:</b> <a href="https://www.linkedin.com/in/danushika-n-323b4628b/">danushika-n</a></td></tr>
  <tr><td colspan="2" align="center"><b>Email:</b> <a href="mailto:danushikanagarajan@gmail.com">danushikanagarajan@gmail.com</a></td></tr>
</table>

Built by **Danushika N**
