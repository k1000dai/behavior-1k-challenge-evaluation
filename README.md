# Omnigibson Docker setup

## Usage
1. set environment variables
`cp .env.example .env`

fix the values in .env

default port is 6098

2. build and run the container
```
docker-compose build
```

```
docker compose up -d
```

4. ssh to the remote server with port forwarding
```
ssh -L 6080:localhost:6098 <remote_server>
```

3. access the web UI at `http://localhost:6080` (or `http://localhost:6080/vnc.html?autoconnect=true&resize=scale` if you want to use VNC)