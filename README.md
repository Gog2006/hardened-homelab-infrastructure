Multi-Tier Homelab: Secure Application Delivery & Monitoring

This repository documents the architecture and deployment of a secure, production-grade homelab environment hosted on Proxmox VE 8.x. The project focuses on bypassing restrictive networking (Double NAT) using Cloudflare Tunnels, implementing a high-availability NGINX Load Balancer, and establishing a full observability stack with Prometheus and Grafana.

Architecture Overview

The infrastructure is designed as a multi-tier system to mimic real-world enterprise environments:
Edge Layer: Cloudflare Edge Network handles DDoS protection and SSL termination.
Access Layer: cloudflared creates an outbound-only tunnel, eliminating the need for open router ports.
Delivery Layer: VM3 (lb-vm) runs NGINX to distribute traffic and enforce A+ security headers.
Application Layer: VM1 (app-vm) hosts the core Flask application, PostgreSQL, and Redis.
Observability Layer: A dedicated monitoring VM scrapes metrics from all nodes for real-time visualization.

 Repository Structure

The project documentation is organized into two primary formats for ease of access:
/docs: Contains original Markdown (.md) files with raw configuration snippets and step-by-step terminal commands.

Key Documentation Highlights:

Cloudflare Tunnel Setup: Bypassing Double NAT and automating SSL/TLS.
NGINX Load Balancer: Configuring upstream servers and achieving an A+ rating on securityheaders.com.
Monitoring Stack: Deploying Prometheus and Grafana with Node Exporter for hardware/OS metrics.

 Security Features

Zero Open Ports: Outbound-only connectivity via Cloudflare Tunnel protects the home IP from exposure.
Hardened SSH: Key-based authentication only; root login and password authentication are strictly disabled.
Traffic Sanitization: NGINX is configured with strict Content Security Policies (CSP) and HSTS (max-age=31536000).
Firewall Isolation: UFW policies restrict internal metrics (Port 9100) to the monitoring instance only.

 Technical Stack

Hypervisor: Proxmox VE 8.x
OS: Ubuntu Server 22.04 LTS
Proxy/LB: NGINX, Cloudflare Tunnel
Monitoring: Prometheus, Grafana, Node Exporter
App Components: Flask, PostgreSQL, Redis, Docker

 How to Use This Repo
 
Plan your Variables: Reference the Variables — Adapt Before Use section in each .md file in the /docs folder.
Deployment Order: 
1. Provision VMs on Proxmox.
2. Set up the Monitoring Stack to track the health of subsequent builds.
3. Configure the NGINX Load Balancer.
4. Establish the Cloudflare Tunnel to bring the services online.
Developed as part of a professional DevOps portfolio to demonstrate expertise in secure infrastructure and automated monitoring.