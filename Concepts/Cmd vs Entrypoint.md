CMD vs ENTRYPOINT in Docker

Both CMD and ENTRYPOINT define what process runs inside your container, but they behave differently.

🔹 CMD

Provides default arguments for a container.

Can be overridden when running the container with docker run ... <command>.

Useful when you want to supply a default, but still allow flexibility.

Example:

FROM ubuntu:20.04
CMD ["echo", "Hello from CMD"]


Running:

docker run myimage
# Output: Hello from CMD


Overriding:

docker run myimage echo "Overridden"
# Output: Overridden

🔹 ENTRYPOINT

Defines the main executable for the container.

Always executed, cannot be completely overridden (only arguments can be appended).

Makes your container act like a single-purpose executable.

Example:

FROM ubuntu:20.04
ENTRYPOINT ["echo", "Hello from ENTRYPOINT"]


Running:

docker run myimage
# Output: Hello from ENTRYPOINT


Appending arguments:

docker run myimage "extra"
# Output: Hello from ENTRYPOINT extra

🔹 ENTRYPOINT + CMD Together

You can combine them:

ENTRYPOINT = the command/executable.

CMD = default arguments for that command.

Example:

FROM ubuntu:20.04
ENTRYPOINT ["echo"]
CMD ["Hello from CMD"]


Running:

docker run myimage
# Output: Hello from CMD


Overriding:

docker run myimage "Hi there!"
# Output: Hi there!

✅ Rule of Thumb

Use CMD when you want a default command that users might override easily.

Use ENTRYPOINT when you want your container to behave like a dedicated binary/executable.

Use ENTRYPOINT + CMD when you want both a fixed command with default arguments.

👉 Do you want me to also show you real-world cases (like nginx, python, etc.) where ENTRYPOINT vs CMD is used differently?

You said:
yes
ChatGPT said:

Perfect 👍 Let’s walk through some real-world Docker images and how they use CMD vs ENTRYPOINT:

🐳 1. Nginx
FROM debian:bullseye-slim
...
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]


ENTRYPOINT → runs the wrapper script /docker-entrypoint.sh (sets up configs, permissions, env vars).

CMD → provides default arguments (nginx -g "daemon off;" keeps Nginx running in foreground).

👉 You can override CMD but not the ENTRYPOINT.

docker run nginx nginx -T   # dumps nginx config instead of running server

🐍 2. Python
FROM debian:bullseye-slim
...
CMD ["python3"]


Only CMD is used.

Defaults to starting Python REPL if no command is provided.

docker run python
# Starts interactive Python shell

docker run python python app.py
# Runs your script instead


👉 Flexible for devs who want to run arbitrary commands.

📦 3. Ubuntu
FROM scratch
...
CMD ["/bin/bash"]


Only CMD.

If you just run docker run -it ubuntu, it drops you into bash.

You can override easily:

docker run -it ubuntu ls /


👉 Designed as a base image, so no strict ENTRYPOINT.

☕ 4. Postgres
FROM debian:bullseye-slim
...
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]


ENTRYPOINT = setup script (init DB, handle env vars).

CMD = default process (postgres).

You can override CMD if you want, e.g.:

docker run postgres pg_isready


👉 This ensures Postgres always initializes correctly, but gives flexibility.

🏗️ 5. Custom App (best practice)

If you build your own microservice:

FROM python:3.11-slim

WORKDIR /app
COPY . .

ENTRYPOINT ["python", "main.py"]
CMD ["--port=8080"]


ENTRYPOINT fixes the main app (python main.py).

CMD supplies default args (--port=8080).

Run normally:

docker run myapp
# Runs: python main.py --port=8080


Override args:

docker run myapp --port=9090
# Runs: python main.py --port=9090


✅ Summary of real-world use

CMD only → base images (Ubuntu, Python).

ENTRYPOINT + CMD → services (Nginx, Postgres, your microservices).

ENTRYPOINT only → when the container must behave as a fixed binary.
