FROM python:3.10.6
WORKDIR /app
COPY requirements.txt /app
RUN python -m pip install --upgrade pip
RUN pip3 install -r requirements.txt 
COPY . .
CMD ["gunicorn", "foodgram.wsgi:application", "--bind", "0:8000" ]
