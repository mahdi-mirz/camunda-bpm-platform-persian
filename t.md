To update a Django project that uses Docker when new code is pushed to GitHub, you can configure the webhook to trigger a Docker rebuild and restart. Here’s how to set it up:

### 1. Set Up GitHub Webhook

Follow the steps to create a webhook in GitHub:
- Go to your Django project repository’s **Settings** on GitHub.
- Under **Webhooks**, click on **Add webhook**.
- Set the **Payload URL** to your server’s endpoint (e.g., `http://your-server-ip/update-server`).
- Choose **Content type** as `application/json`.
- Select **Just the push event** to trigger the webhook when you push code.

### 2. Create a Webhook Receiver on Your Server

On your Django server, set up a Flask app (or a similar lightweight server) to handle the webhook and trigger a Docker update:

1. **Install Flask**:

   ```bash
   pip install flask
   ```

2. **Write the Webhook Receiver Script**:

   Create a Python file (e.g., `webhook_listener.py`) on your server with the following code:

   ```python
   from flask import Flask, request
   import subprocess

   app = Flask(__name__)

   @app.route('/update-server', methods=['POST'])
   def webhook():
       data = request.json
       if data and data['ref'] == 'refs/heads/main':  # assuming main branch
           # Pull the latest code
           subprocess.call(['git', 'pull', 'origin', 'main'])
           
           # Stop and remove the existing Docker container
           subprocess.call(['docker', 'compose', '-f', 'docker-compose.yml', 'down'])

           # Rebuild the Docker image
           subprocess.call(['docker', 'compose', '-f', 'docker-compose.yml', 'build'])

           # Start the updated Docker container
           subprocess.call(['docker', 'compose', '-f', 'docker-compose.yml', 'up', '-d'])

       return '', 200

   if __name__ == '__main__':
       app.run(port=5000)
   ```

3. **Run the Webhook Listener**:

   Run this Flask app to listen for webhook requests:

   ```bash
   python webhook_listener.py
   ```

   Use **screen**, **tmux**, or **systemd** to keep this listener running in the background.

### 3. Docker Compose Configuration

Make sure your `docker-compose.yml` file is correctly set up to build and run your Django application. Here’s a basic example of what your `docker-compose.yml` might look like:

```yaml
version: '3.8'

services:
  web:
    build: .
    command: gunicorn your_django_project.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db

  db:
    image: postgres
    environment:
      POSTGRES_DB: your_db_name
      POSTGRES_USER: your_db_user
      POSTGRES_PASSWORD: your_db_password
```

Adjust the settings as per your Django and database setup.

### 4. Securing the Webhook Endpoint (Optional)

To secure the webhook endpoint, you could set up a **secret token** in GitHub’s webhook settings. Then, in your webhook listener script, you can validate incoming requests to ensure they’re from GitHub.

This setup will pull the latest changes, rebuild your Docker container, and restart the Django application in Docker whenever you push new code. Let me know if there’s anything more specific you’d like help with!












# New 001
A `400 Bad Request` error typically indicates that there’s an issue with the request format or how it’s being received. This often happens when GitHub sends a POST request that your Flask webhook receiver is not handling as expected.

Here are some common issues and solutions:

### 1. Check Payload Type Compatibility

By default, GitHub sends JSON payloads in webhooks. Make sure your webhook listener expects JSON data.

In your Flask script, ensure you’re properly checking and handling the JSON payload:

```python
from flask import Flask, request, abort
import subprocess

app = Flask(__name__)

@app.route('/update-server', methods=['POST'])
def webhook():
    if request.headers.get('Content-Type') == 'application/json':
        data = request.json  # Get JSON data from request
    else:
        abort(400, "Invalid content type")

    # Check for correct branch, assuming main
    if data and data.get('ref') == 'refs/heads/main':
        try:
            # Pull the latest code
            subprocess.call(['git', 'pull', 'origin', 'main'])
           
            # Stop and remove the existing Docker container
            subprocess.call(['docker', 'compose', '-f', 'docker-compose.yml', 'down'])

            # Rebuild the Docker image
            subprocess.call(['docker', 'compose', '-f', 'docker-compose.yml', 'build'])

            # Start the updated Docker container
            subprocess.call(['docker', 'compose', '-f', 'docker-compose.yml', 'up', '-d'])
        except Exception as e:
            abort(500, f"Deployment failed: {str(e)}")
    else:
        abort(400, "No data or incorrect branch")

    return 'Success', 200

if __name__ == '__main__':
    app.run(port=5000)
```

### 2. Check GitHub Webhook Settings

In the GitHub repository's settings:
- Ensure the **Content-Type** is set to `application/json`.
- Ensure you’re using the correct **Payload URL** (e.g., `http://your-server-ip:5000/update-server`).

### 3. Check Server Logs for Detailed Errors

Flask should output specific errors to the console if there’s an issue with receiving the data. Run your Flask app in a terminal or check logs if using a process manager (e.g., systemd or supervisor) to catch detailed error messages.

### 4. Debug with Simple Logging (Optional)

Add some logging to verify what’s coming into your webhook:

```python
import logging
logging.basicConfig(level=logging.INFO)

@app.route('/update-server', methods=['POST'])
def webhook():
    if request.headers.get('Content-Type') == 'application/json':
        data = request.json
        logging.info("Received data: %s", data)  # Log incoming data for debugging
    else:
        abort(400, "Invalid content type")

    # (Rest of your code remains the same)
```

After making these changes, test pushing to your GitHub repository and check if the 400 error persists. Let me know if this resolves it or if you encounter other errors!
