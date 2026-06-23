# We use a specific version tag, never 'latest'
# This is enforced later by OPA Gatekeeper — our policy will BLOCK
# any deployment using the latest tag
FROM python:3.12-slim

# Run as non-root user — security best practice
# OPA will also enforce this at admission
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

USER appuser

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]