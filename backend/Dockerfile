# imagen oficial de python con version fija (nunca latest)
FROM python:3.12-slim

WORKDIR /app

# primero las dependencias para aprovechar la cache de capas
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

# usuario sin privilegios en vez de root
RUN useradd --no-create-home appuser
USER appuser

EXPOSE 8000

# gunicorn escuchando en todas las interfaces y sirviendo el objeto app de app.py
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
