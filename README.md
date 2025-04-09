# mlx-server-OAI-compat

## Description
This repository hosts a high-performance proxy server designed to be fully compatible with the OAI-compat protocol, specifically tailored for MLX models. Developed using Python and powered by the FastAPI framework, it provides an efficient, scalable, and user-friendly solution for handling MLX-based vision and language model inference requests.

> **Note:** This project currently supports **MacOS with M-series chips** only as it specifically leverages MLX, Apple's framework optimized for Apple Silicon.

## Installation

Follow these steps to set up the MLX-powered server:

### Native Python Recommendation
We recommend using a native Python version (supporting ARM architecture). The development environment for this project uses Python `3.11`.

### Setup Steps
1. Create a virtual environment for the project:
    ```bash
    python3 -m venv oai-compat-server
    ```

2. Activate the virtual environment:
    ```bash
    source oai-compat-server/bin/activate
    ```

3. Install the required Python dependencies:
    ```bash
    pip install -e .
    ```

### Troubleshooting
**Issue:** My OS and Python versions meet the requirements, but `pip` cannot find a matching distribution.

**Cause:** You might be using a non-native Python version. Run the following command to check:
```bash
python -c "import platform; print(platform.processor())"
```
If the output is `i386` (on an M-series machine), you are using a non-native Python. Switch to a native Python version. A good approach is to use [Conda](https://stackoverflow.com/questions/65415996/how-to-specify-the-architecture-or-platform-for-a-new-conda-environment-apple).

## Usage
To start the MLX server, activate the virtual environment and run the main application file:
```bash
source oai-compat-server/bin/activate
python -m app.main \
  --model-path <path-to-mlx-model> \
  --max-concurrency 1 \
  --queue-timeout 300 \
  --queue-size 100
```

Parameters:
- `--model-path`: Path to the MLX model directory (supports mlx models only)
- `--max-concurrency`: Maximum number of concurrent requests (default: 1)
- `--queue-timeout`: Request timeout in seconds (default: 300)
- `--queue-size`: Maximum queue size for pending requests (default: 100)
- `--port`: Port to run the server on (default: 8000)
- `--host`: Host to run the server on (default: 0.0.0.0)

Example:
```bash
python -m app.main \
  --model-path mlx-community/gemma-3-4b-it-4bit \
  --max-concurrency 1 \
  --queue-timeout 300 \
  --queue-size 100
```

## CLI Usage
You can also install the package and use the CLI command to launch the server:

### Installation with CLI
```bash
pip install -e .
```
or
```bash
pip install git@https://github.com/cubist38/mlx-server-OAI-compat.git
```

### Using the CLI
```bash
mlx-server launch --model-path <path-to-mlx-model> --port 8000
```

All parameters available in the Python version are also available in the CLI:
```bash
mlx-server launch \
  --model-path mlx-community/gemma-3-4b-it-4bit \
  --port 8000 \
  --max-concurrency 1 \
  --queue-timeout 300 \
  --queue-size 100
```

### Getting Help
```bash
mlx-server --help
mlx-server launch --help
```

### Checking Version
```bash
mlx-server --version
```

## Request Queue System

The server implements a robust request queue system to prevent overloading the MLX model and ensure fair processing of requests.

### Key Features

- **Concurrency control**: Limits the number of simultaneous MLX model inferences
- **Queuing**: Handles pending requests in a fair, first-come-first-served manner
- **Timeout handling**: Automatically fails requests that exceed the configured timeout
- **Status monitoring**: Provides endpoints to monitor queue status

### Architecture

The queue system consists of two main components:

1. **RequestQueue**: A generic async queue implementation that:
   - Manages a queue of pending requests
   - Controls concurrent execution with a semaphore
   - Handles timeouts and errors
   - Provides statistics about queue status

2. **MLXHandler integration**: The service maintains dedicated queues for efficient processing of MLX model requests (both vision and text)

### Monitoring

The service provides an endpoint to monitor queue statistics:

```bash
curl http://localhost:8000/v1/queue/stats
```

Response example:

```json
{
  "status": "ok",
  "queue_stats": {
    "vision_queue": {
      "running": true,
      "queue_size": 3,
      "max_queue_size": 100,
      "active_requests": 5,
      "max_concurrency": 2
    }
  }
}
```

### Error Handling

When the queue is full, the server returns a 429 error:

```json
{
  "detail": "Too many requests. Service is at capacity."
}
```

When a request times out, an exception is raised in the client's response.

### Implementation Notes

- Streaming requests are handled efficiently to leverage MLX's generation capabilities
- Both vision and text requests are fully supported through the MLX interface
- Each request gets a unique ID for tracking and debugging
- Queue statistics are updated in real-time
- Model loading is optimized for MLX's architecture

## Performance Monitoring

The server includes comprehensive performance monitoring and benchmarking capabilities to help track and optimize MLX model performance.

### Key Features

- **Token Per Second (TPS) Tracking**: Real-time monitoring of MLX model generation speed
- **Detailed Request Metrics**: Per-request statistics including token counts, word counts, and processing time
- **Historical Performance Data**: Maintains history of recent requests for trend analysis
- **Request Type Breakdown**: Separate metrics for different types of requests (vision/text, streaming/non-streaming)
- **Robust Error Handling**: Fault-tolerant metrics collection with fallbacks for missing data

### Metrics Endpoint

The `/v1/queue/stats` endpoint now provides enhanced performance metrics:

```bash
curl http://localhost:8000/v1/queue/stats
```

Response example:

```json
{
  "status": "ok",
  "queue_stats": {
    "running": true,
    "queue_size": 3,
    "max_queue_size": 100,
    "active_requests": 5,
    "max_concurrency": 2
  },
  "metrics": {
    "total_requests": 100,
    "total_tokens": 5000,
    "total_time": 50.5,
    "request_types": {
      "vision": 40,
      "vision_stream": 20,
      "text": 30,
      "text_stream": 10
    },
    "error_count": 2,
    "performance": {
      "avg_tps": 99.0,
      "max_tps": 150.0,
      "min_tps": 50.0,
      "recent_requests": 100
    }
  }
}
```

### Metrics System Design

The performance metrics system is designed with robustness in mind:

1. **Consistent Metrics Keys**: All request handlers use a standardized format for metrics
2. **Fault Tolerance**: The system uses fallbacks when processing metrics data:
   ```python
   # Using get() with defaults for safety
   self.metrics["total_tokens"] += metrics.get("token_count", metrics.get("estimated_tokens", 0))
   ```
3. **Key Normalization**: The system automatically maps between different key formats:
   - `estimated_tokens` from token estimator
   - `token_count` for metrics system
4. **Error Resilience**: The metrics system continues functioning even if one component fails

### Metrics Details

The performance metrics include:

- **Request Statistics**:
  - Total number of requests processed
  - Breakdown by request type (vision/text, streaming/non-streaming)
  - Total tokens generated
  - Total processing time

- **Performance Metrics**:
  - Average Tokens Per Second (TPS)
  - Maximum TPS observed
  - Minimum TPS observed
  - Number of recent requests in history

- **Error Tracking**:
  - Total number of errors encountered
  - Error rate calculation

### Token Estimation

The server uses sophisticated token estimation methods:

- Word-based estimation (words/1.3)
- Character-based estimation (chars/4)
- Combined average for improved accuracy

This provides more accurate performance metrics for benchmarking and optimization.

### Logging

Detailed performance logs are available in the server logs:

```
Request completed: vision_stream
Tokens: 150 (words: 75, chars: 450)
Time: 1.50s
TPS: 100.00
Avg TPS: 95.50
```

## API Usage

### Text Request Example

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": "What is the capital of France?"
      }
    ]
  }'
```

### Vision Request Example
You can make vision requests to analyze images using the `/v1/chat/completions` endpoint. Here's an example:

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "What is in this image?"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"
            }
          }
        ]
      }
    ] 
  }'
```

### Request Format
- `messages`: Array of message objects containing:
  - `role`: The role of the message sender ("user", "assistant", or "system")
  - `content`: For vision requests, an array of content objects:
    - `type`: Either "text" or "image_url"
    - `text`: The text prompt (for type "text")
    - `image_url`: Object containing the image URL (for type "image_url")
- `stream`: Optional boolean to enable streaming responses
- Additional parameters: `temperature`, `max_tokens`, `top_p`, etc.

### Response Format
The server will return responses in OpenAI-compatible format:

```json
{
  "id": "chatcmpl-1234567890",
  "object": "chat.completion",
  "created": 1234567890,
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "The image shows a wooden boardwalk..."
      },
      "finish_reason": "stop"
    }
  ]
}
```

### Streaming Responses
For streaming responses, add `stream": true` to your request. The response will be in Server-Sent Events (SSE) format:

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "stream": true,
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "What is in this image?"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://example.com/image.jpg"
            }
          }
        ]
      }
    ] 
  }'
```

### Multi-turn Conversations
The API supports multi-turn conversations with images. You can include previous messages in the history:

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant that describes images."
    },
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "What is in this image?"
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "https://example.com/image.jpg"
          }
        }
      ]
    },
    {
      "role": "assistant",
      "content": "The image shows a wooden boardwalk..."
    },
    {
      "role": "user",
      "content": "Are there any people in the image?"
    }
  ]
}
```

### Image Token Handling

For multimodal models that require specific image token formatting, the server handles this automatically. The implementation uses the following approach:

```python
def handle_list_with_image(prompt, role, num_images, skip_image_token=False):
    """Format message with proper image token handling"""
    content = [{"type": "text", "text": prompt}]
    if role == "user" and not skip_image_token:
        content.extend([{"type": "image"}] * num_images)
    return {"role": role, "content": content}
```

This ensures that:
- Image tokens are automatically added to user messages
- You can control whether image tokens appear before or after the text
- The server handles both single-turn and multi-turn conversations with images

## Contributing
We welcome contributions to improve this project! Here's how you can contribute:
1. Fork the repository to your GitHub account.
2. Create a new branch for your feature or bug fix:
    ```bash
    git checkout -b feature-name
    ```
3. Commit your changes with clear and concise messages:
    ```bash
    git commit -m "Add feature-name"
    ```
4. Push your branch to your forked repository:
    ```bash
    git push origin feature-name
    ```
5. Open a pull request to the main repository for review.

## License
This project is licensed under the [MIT License](LICENSE). You are free to use, modify, and distribute it under the terms of the license.

## Support
If you encounter any issues or have questions, please:
- Open an issue in the repository.
- Contact the maintainers via the provided contact information.

Stay tuned for updates and enhancements!

## Acknowledgments

We extend our heartfelt gratitude to the following individuals and organizations whose contributions have been instrumental in making this project possible:

### Core Technologies
- [MLX team](https://github.com/ml-explore/mlx) for developing the groundbreaking MLX framework, which provides the foundation for efficient machine learning on Apple Silicon
- [mlx-vlm](https://github.com/Blaizzy/mlx-vlm/tree/main) for pioneering multimodal model support within the MLX ecosystem
- [mlx-community](https://huggingface.co/mlx-community) for curating and maintaining a diverse collection of high-quality MLX models

### Open Source Community
We deeply appreciate the broader open-source community for their invaluable contributions. Your dedication to:
- Innovation in machine learning and AI
- Collaborative development practices
- Knowledge sharing and documentation
- Continuous improvement of tools and frameworks

Your collective efforts continue to drive progress and make projects like this possible. We are proud to be part of this vibrant ecosystem.

### Special Thanks
A special acknowledgment to all contributors, users, and supporters who have helped shape this project through their feedback, bug reports, and suggestions. Your engagement helps make this project better for everyone.

## Recent Updates

### Enhanced User Interface
The server now features a more visually appealing command-line interface with:
- ✨ Beautiful startup banners with emoji indicators
- 📏 Clean and consistent horizontal dividers
- 📊 Enhanced logging with improved readability
- 🎨 Color-coded output for better visual distinction

### Improved Architecture
Recent architectural improvements include:
- 🔄 Refactored configuration management to support cleaner dependency injection
- 🛡️ Enhanced error handling with proper context management
- 🧩 Factory pattern implementation for more flexible component creation
- ⚙️ Optimized request processing pipeline

### Warning Suppression
The server now automatically suppresses common warnings:
- 🔕 Transformers framework detection warnings
- 📝 Explicit configuration for Python warnings system
- 🎯 More targeted filtering of specific warning messages

### Enhanced Error Resilience
- 🔄 Improved recovery from initialization errors
- 🛑 Graceful handling of configuration issues
- 🔄 Better state management during startup and shutdown

These improvements enhance both user experience and system reliability, making the server easier to use and more robust in production environments.