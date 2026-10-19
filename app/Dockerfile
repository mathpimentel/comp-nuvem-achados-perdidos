FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

RUN groupadd appuser \
    && useradd --gid appuser --no-create-home appuser \
    && mkdir -p /app/uploads \
    && chown appuser:appuser /app/uploads

COPY --chown=appuser:appuser app.py .

USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--access-logfile", "-", "app:app"]
