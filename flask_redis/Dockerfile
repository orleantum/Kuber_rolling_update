FROM python:3.10-alpine

WORKDIR /code

COPY requirements.txt requirements.txt

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

COPY . .

CMD ["python", "app.py"]
