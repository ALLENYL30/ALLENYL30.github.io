+++
author = "yuhao"
title = "Implementing Apollo Configuration in .NET Core"
date = "2024-08-24"
description = "Apollo is a distributed configuration management system developed by Ctrip's Framework Department. It enables centralized configuration management across multiple environments and clusters,..."
tags = [
    ".NET",
    ".NET Core",
    "Apollo",
]
categories = [
    "Programming/Development",
]
+++
# Apollo Configuration Center Guide

## Overview

Apollo is a **distributed configuration management system** developed by Ctrip's Framework Department. It enables centralized configuration management across multiple environments and clusters, supporting real-time configuration updates with robust permission control and process governance. Ideal for microservices architecture, Apollo's server is built on Spring Boot/Cloud and runs out-of-the-box without requiring additional containers like Tomcat.

**GitHub**: [https://github.com/ctripcorp/apollo](https://github.com/ctripcorp/apollo)

## Key Features

- Centralized configuration management for multi-environment/multi-cluster scenarios
- Real-time configuration delivery
- Granular permission control & audit trails
- Configuration versioning & rollback
- Client configuration cache for high availability
- Multi-language client support (Java, .NET, etc.)

> This guide focuses on .NET Core integration. For product overview, see [Apollo Configuration Center Introduction](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E4%BB%8B%E7%BB%8D).

## Deployment Options

### 1. Quick Start (Docker)

```bash
# Download deployment files
wget https://github.com/ctripcorp/apollo/raw/master/scripts/docker-quick-start/docker-compose.yml
wget https://github.com/ctripcorp/apollo/raw/master/scripts/docker-quick-start/apollo-env.properties

# Start services
docker-compose up
```

Verify successful startup:

```bash
docker logs apollo-quick-start | grep 'Config service started'
# Expected output: "Config service started. Visit http://localhost:8080"
```

Access:

- Portal: [http://localhost:8070](http://localhost:8070)

