# Docker configurations template for django projects includes nginx, redis, celery, postgresql.

This repository is to contain a ready to go django application for servers.

## Get Started

### Download docker first!

Download docker from it's official website! https://www.docker.com/

### Build your containers
This command will download a group of images for once and build your group of containers. 

``` bash
docker-compose build
```

### Run your container with creating your template
Run your docker and delete it afterwards. We do this line for creating and mirroring essential files.

``` bash
docker-compose run --rm app django-admin startproject core ./
```

### Now you can just start your docker container
At this point your containers will run all together but your celery will give an error and will try to run itself continously.

``` bash
docker-compose up
```

If you want celery on your project head over [here](#celery-setup).

If not just comment out or completely delete celery service from your container-compose.yml file.


<hr>
<br>

> You can see or exec your docker containers on your docker desktop application.

# Setups

Following setups are just examples, they are not perfect and may not be suitable for your project. I forget stuff easly so they just help me out what is what with a quick glance. :P

## Setup Redis

Here is a quick redis setup guide, you can do it easly do it from here in case you want to.

### 1. Create a new app for redis.
Exec in your docker container from docker desktop or just run this command to create a new application in django project for redis.

``` bash
docker-compose run --rm app django-admin startapp index
```

> "app" is your django service running you can see it from "docker-compose.yml" file.

### 2. Create and add files in your new application.

You can add your "consumers.py" and "router.py" inside "index" (your new application name in your case)

Here is a base example for consumers.py and router.py:

``` py
# consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class NotificationConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.group_name = 'notifications'
        
        # Join group
        await self.channel_layer.group_add(
            self.group_name,
            self.channel_name
        )

        await self.accept()

    async def disconnect(self, close_code):
        # Leave group
        await self.channel_layer.group_discard(
            self.group_name,
            self.channel_name
        )

    # Receive message from WebSocket
    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        event = text_data_json['event']
        username = text_data_json['username']

        # Send message to group
        await self.channel_layer.group_send(
            self.group_name,
            {
                'type': 'user_event',
                'event': event,
                'username': username,
            }
        )

    # Receive message from group
    async def user_event(self, event):
        await self.send(text_data=json.dumps({
            'event': event['event'],
            'username': event['username']
        }))
```

``` py
# routing.py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/notifications/$', consumers.NotificationConsumer.as_asgi()),
]
```

### 3. Edit your settings.py.
Add 'channels' into your django_applications in settings.py.

and

Add these config code on your settings.py.

``` py
ASGI_APPLICATION = 'core.asgi.application'

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        "CONFIG": {
            'hosts': [('127.0.0.1', 6379)],
        }
    }
}
```

### 4. Add ASGI application.

Change your asgi.py file in core folder with this.

``` py
import os

from django.core.asgi import get_asgi_application

from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
import index.routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'core.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(
            index.routing.websocket_urlpatterns
        )
    ),
})
```

With all these you should be ready to go for redis side of the things.

## Setup Postgresql

When you run your docker container for the first time, it actually creates a database for u already. Now all we have to do is just configure.

### Edit your settings.py

We will be changing our default DATABASE config into this;

``` py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'PASSWORD': 'postgres',
        'HOST': 'db',
        'PORT': 5432,
    }
}
```

After this test your database with;

``` bash
docker-compose run --rm app python manage.py migrate
```

After this line there should be no errors. Well know you are good to go with postgresql too.

## <a name="celery-setup"></a>  Setup Celery 

Now we will add these lines into our celery.py file in core folder.

``` py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'core.settings')
app = Celery('core')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

Afterwards we will edit _init_.py in core folder.

``` py
from __future__ import absolute_import, unicode_literals
from .celery import app as celery_app

__app__ = ('celery_app',)
```

And add these lines on settings.py

``` py
CELERY_BROKER_URL = 'redis://redis:6379'
CELERY_RESULT_BACKEND = 'redis://redis:6379'
```

Now lets add task.py inside index folder.

``` py
from __future__ import absolute_import, unicode_literals

from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```
> Do not forget to add your index application in your settings.py file.

You can test celery in manage.py shell

```
python manage.py shell
from index.tasks import add
task = add.delay(2, 2)
```

<hr>