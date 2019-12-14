---
layout: post
title: "The villager-or-not web application reveal"
author: "Joakim Lönnegren"
categories: blog
tags: [Age of Empires 2, Python, Docker, Flask]
image:
  feature: villager-or-not-webapp/feature.png
  teaser: villager-or-not-webapp/teaser.png
---

If you've ever wanted to host your own [Villager-or-not]() app you're in luck! I understand this might be a niche interest, though. Anyway, I've put it up on [Docker hub](https://hub.docker.com/r/joalon/villager-or-not) and I've put the source code on [Github](https://github.com/joalon/aoe2-villager-or-not) (under the webapp/ dir). If you want to run it right now and you have docker installed, you can try this: `docker run -d -p 8080:8080 joalon/villager-or-not`.

The GUI isn't really polished yet, here's a preview in all its glory:

![Villager or not](/images/villager-or-not-webapp/running-locally.png)

## Project setup
The project is a Flask backend running under [uwsgi](https://uwsgi-docs.readthedocs.io/) and [Nginx](https://www.nginx.com/) which is supervised by [supervisord](http://supervisord.org/). Quite a mouthful! This is how the file structure looks:

```fish
villager-or-not/
     ...
     webapp/
         Dockerfile
         Makefile
         export.pkl
         nginx.conf
         supervisord.conf
         uwsgi.ini
         requirements.txt
         villager-or-not.conf
         app/
         templates/
```

Where export.pkl was generated as a step in my earlier blog post about [training a nerual network](https://joalon.se/blog/Villager-or-not.html).

I also wrote a Makefile to simplyfy testing. It can execute the `run`, `build` and `clean` verbs.

## Flask api
Here's a shortened version of the Flask api which takes a file in a HTTP Post method and applies it to the machine learning model.

```python3
@app.route('/', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        file = request.files['file']
        if file and allowed_file(file.filename):
            image = open_image(file)
            prediction = str(learn.predict(image)[0])
            if prediction == 'villager':
                flash('It\'s a villager!')
            else:
                flash('It\'s not a villager!')
    return render_template('upload_image.html')
```

The front end is a simple template, the interesting part is in `upload_file.html`:

{% raw %}
```
{% extends "layout.html" %}
{% block body %}
<div align="center">
    <h1>Upload new File</h1>
    <form method=post enctype=multipart/form-data>
        <p><input type=file name=file>
        <input type=submit value=Upload>
    </form>
</div>
{% endblock %}
```
{% endraw %}

## Nginx configuration
To get the nginx + uwsgi + supervisord configuration working serving the Flask api I followed the steps in [this tutorial](https://medium.com/@gabimelo/developing-a-flask-api-in-a-docker-container-with-uwsgi-and-nginx-e089e43ed90e) by Gabriela Melo. Here's how it is setup in the dockerfile:

```Dockerfile
FROM python:3.7

RUN apt update &&\
    apt install -y supervisor nginx &&\
    pip3 install uwsgi

RUN useradd --no-create-home nginx
RUN rm /etc/nginx/sites-enabled/default

WORKDIR /opt/villager-or-not

COPY supervisord.conf /etc/supervisor/
COPY uwsgi.ini /etc/uwsgi/
COPY nginx.conf /etc/nginx/
COPY villager-or-not.conf /etc/nginx/conf.d/

COPY ./requirements.txt .
COPY ./export.pkl .
COPY ./templates ./app/templates

RUN pip3 install -r requirements.txt

COPY ./app/* /opt/villager-or-not/app/

CMD ["/usr/bin/supervisord"]
```

## Summary
Check it out by running `docker run -d -p 8080:8080 joalon/villager-or-not`. I'll eventually host it for the curious at [https://joalon.se/apps/villager-or-not/](https://joalon.se/apps/villager-or-not/) as well.

