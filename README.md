Real-Time Chat App — Dockerized Deployment with NGINX & CI/CD
1. Project Overview

This project is a real-time WebSocket chat application, containerized with Docker and deployed behind an NGINX reverse proxy on an AWS EC2 instance. The application was provided with a deliberately broken deployment configuration; this repository contains the debugged, working setup along with an automated CI/CD pipeline using GitHub Actions.

Stack:

Backend: Python FastAPI, WebSocket support on /ws
Frontend: Static single-page HTML client
Reverse proxy: NGINX (serves frontend + proxies WebSocket traffic)
Orchestration: Docker Compose
Deployment: AWS EC2 (free tier)
CI/CD: GitHub Actions (auto-deploy on push to main)

Live URL: http://<YOUR_EC2_PUBLIC_IP>
