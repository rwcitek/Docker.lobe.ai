
# lobe.ai with Docker
The thought was to create a Docker container that contain everything needed, including example training images, to demonstrate how to use lobe.ai and its pythong SDK.
Furthermore, the Docker image could be then be used as a base from which to do further training and classifying.

These are notes towards that goal.  They are likely specific to being on a Mac and, specifically, to my configuration on that Mac.

## Extracting the entire build envronment from the image on DockerHub
The Docker image contains all the lobe and dependent software packages, sample training images that I used with Lobe, sample test images, the exported TensorFlow model from Lobe, and a python script that classifies an image.

### Copy the entire development environment from the Docker image to the host
```bash
cd $( mktemp --directory /tmp/lobe.ai.XXXXXX ) &&
docker run --rm -v "${PWD}":/output rwcitek/lobe.ai \
  rsync -vaR --progress /lobe.ai/./dental/ /output/
```

Here's a docker command to run the python code that classifies a JPG image.  The classifier identifies an image as either toothpaste or dental floss.

### to run the python code
```bash
docker run --rm -i rwcitek/lobe.ai python3 /lobe.ai/dental/code/dental.py
```








```bash
cd ~/Desktop/zzz/lobe.ai/dental.test

URL=http://localhost:38101/v1/predict/f3f0b084-4800-49d4-bd92-1acfd264d68b
URL=http://192.168.0.4:38101/v1/predict/9c035df1-737e-4d64-bbf8-7aba56ec1234
find test -name '*.jpg' | sort |
while read img ; do
  echo == ${img}
  { cat <<eof
{
  "image": "$( base64 -i ${img} )"
}
eof
  } | curl -s -H "Content-Type: application/json" -X POST --data-binary @- ${URL}
  echo
done |
tee prediction.yaml
```

## Python setup

### launch container
```bash
lobe=~/Desktop/zzz/lobe.ai/dental.test
docker run --rm -v ${lobe}:/lobe --name lobe ubuntu sleep inf &
```

### configure container
```bash
docker exec -i lobe /bin/bash <<'eof'
  # Update index
  apt-get update

  # Install usability packages
  apt-get install -y git libheif-examples tree less rsync vim

  # Install Python3
  apt-get install -y python3-dev python3-pip git

  # Install Pillow dependencies
  apt-get install -y libatlas-base-dev libopenjp2-7 libtiff5 libjpeg62-dev

  # Install lobe-python
  pip install setuptools
  pip install lobe[all]
eof
```

### create python script
```bash
docker exec -i lobe /bin/bash <<'eof1'
  <<'eof' cat > /tmp/dental.py
from lobe import ImageModel

model = ImageModel.load('/lobe/dental TensorFlow/')

# Predict from an image file
result = model.predict_from_file('/lobe/floss/IMG_4880.jpeg')

# Print top prediction
print(result.prediction)

# Print all classes
for label, confidence in result.labels:
    print(f"{label}: {confidence*100}%")

eof

  # test script within container
  python3 /tmp/dental.py
eof1
```

### Stop and remove container
```bash
docker container stop lobe 
```


## Create a lobe.ai Docker image with images and code
```bash
# Create the dental.py Python script
<<'eof' cat > dental.py
from lobe import ImageModel
import magic
import sys

images = sys.argv[1:] if len( sys.argv ) > 1 else ['/lobe.ai/dental/test/toothpaste/toothpaste-01.jpg']

model = ImageModel.load('/lobe.ai/dental/test/dental TensorFlow')

for image in images:
  # Print image name
  print(image)
  
  # Predict from an image file
  try:
    result = model.predict_from_file(image)
  except:
    print(f"An exception occurred:\n{ sys.exc_info()[0] }")
    print(f"File type:\n{ magic.from_file(image) }")
    print()
    continue
  
  # Print top prediction
  print(result.prediction)
  
  # Print all classes
  for label, confidence in result.labels:
      print(f"{label}: {confidence*100}%")

  print()

eof

# Create the Dockerfile
{ cat <<'eof'
FROM ubuntu:20.04

MAINTAINER Robert Citek robert.citek@gmail.com

RUN mkdir -p /lobe.ai/dental/test
ADD ./test/ /lobe.ai/dental/test/

RUN mkdir -p /lobe.ai/dental/training
ADD ./training/ /lobe.ai/dental/training/

RUN mkdir -p /lobe.ai/dental/code
ADD ./dental.py /lobe.ai/dental/code/

RUN  apt-get update && \
     DEBIAN_FRONTEND=noninteractive \
      apt-get install -y \
       curl \
       git \
       libheif-examples \
       tree \
       less \
       rsync \
       vim \
       \
       python3-dev \
       python3-pip \
       \
       libatlas-base-dev \
       libopenjp2-7 \
       libtiff5 \
       libjpeg62-dev

RUN  pip install \
      python-magic \
      setuptools \
      lobe[all]

COPY ./Dockerfile /

eof
} > Dockerfile &&
# Build the image from the Dockerfile
docker build --tag rwcitek/lobe.ai:latest . &&
# Tag the image with a version using the date
docker tag rwcitek/lobe.ai:latest rwcitek/lobe.ai:$( date +%Y-%m-%d-%H-%M )
```

Here's a docker command to run the python code that classifies a JPG image.  The classifier identifies an image as either toothpaste or dental floss.

### to run the python code
```bash
docker run -i --rm rwcitek/lobe.ai python3 /lobe.ai/dental/code/dental.py
```

## Extracting the entire build envronment from the image on DockerHub
The Docker image contains all the lobe and dependent software packages, sample training images that I used with Lobe, sample test images, the exported TensorFlow model from Lobe, and a python script that classifies an image.

### copy the entire development environment from the Docker image to the host
```bash
cd $( mktemp --directory /tmp/lobe.ai.XXXXXX ) &&
docker run -v "${PWD}":/output rwcitek/lobe.ai \
  rsync -vaR --progress /lobe.ai/./dental/ /output/
```

Alternatively, you can copy a subset of the files/folders from the Docker image onto the host.

### copy the training images from the Docker image to the host
```bash
cd $( mktemp --directory /tmp/lobe.ai.XXXXXX ) &&
docker run -v "${PWD}":/output rwcitek/lobe.ai \
  rsync -vaR /lobe.ai/./dental/training/ /output/
```

### copy the test images from the Docker image to the host
```bash
cd $( mktemp --directory /tmp/lobe.ai.XXXXXX ) &&
docker run -v "${PWD}":/output rwcitek/lobe.ai \
  rsync -vaR /lobe.ai/./dental/test/ /output/
```

### copy the Python code from the Docker image to the host
```bash
cd $( mktemp --directory /tmp/lobe.ai.XXXXXX ) &&
docker run -v "${PWD}":/output rwcitek/lobe.ai \
  rsync -vaR /lobe.ai/./dental/code/ /output/
```

You can use Docker's volume option ( -v ) to mask files/folders in the Docker image with files/folders from your host.

### use alternate python script
```bash
docker run -i --rm \
  -v "${PWD}/code/dental.py":"/lobe.ai/dental/code/dental.py" \
  rwcitek/lobe.ai python3 /lobe.ai/dental/code/dental.py
```

# use alternate tensor flow
```bash
docker run -i --rm \
  -v "${PWD}/dental TensorFlow/":"/lobe.ai/dental/training/dental TensorFlow/" \
  rwcitek/lobe.ai python3 /lobe.ai/dental/code/dental.py
```

### using lobe.ai's web API
The URL will change with each instance project started within Lobe.ai.
```bash
{ cat << 'eof2'
 URL=http://192.168.0.4:38101/v1/predict/9c035df1-737e-4d64-bbf8-7aba56ec1234
 find /lobe.ai/dental/test -name '*.jpg' | sort |
 head -1 |
while read img ; do

{ cat <<eof
{
  "image": "$( base64 -w0 ${img} )"
}
eof
} | curl -s -H "Content-Type: application/json" -X POST --data-binary @- ${URL}
done

eof2
} | docker run -i --rm rwcitek/lobe.ai | jq .
```

### clean up and test
```bash
docker builder prune -a
docker run -i --rm rwcitek/lobe.ai date
```
