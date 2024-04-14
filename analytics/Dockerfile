# use python 3.9 image for build docker
FROM python:3.9

# Use folder app as working directory
WORKDIR /app

# Copy file requirement.txt to application folder
COPY /requirements.txt /app/requirements.txt

# Install python dependencies for application
RUN pip install -r requirements.txt

# Copy python source code to application folder
COPY . /app

# Run flask application
ENTRYPOINT [ "python", "app.py" ] 
