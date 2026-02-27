+++
title = "A Minimal Wayland Client in C: Creating a Window from Scratch"
date = 2026-02-27
description = "A simple wayland client"
draft = false

[extra]
toc = true

[taxonomies] 
tags = ["linux","wayland"]
+++

## What is wayland?

Wayland is a modern display server protocol used on Linux systems. It defines how graphical applications (clients) communicate with a
display server called a compositor. Applications render their own graphics and send the finished images (buffers) to the compositor. The
compositor then combines these images and displays the final result on the screen, while also handling input from the keyboard and
mouse.[For More Info...](https://wayland.freedesktop.org/)

## What I am actually building

So I will be building a very simple wayland client from scratch and make a window appear thats it , nothing fancy or advance.  
A rough overview of what I will do

- connect to the compositor
- discover global objects via the registry
- create a surface using xdg-shell
- allocate a shared memory buffer
- draw a solid white window

So lets dive in..

## Connecting to the compositor and binding to global objects

Wayland is a async by nature , so to know the capabilities of a compositor on which i will be working on , we need to ask the compositor
itself to list all its global objects.  
This way a dev can know what capabilities a compositor have and what it doesn't. All this is done through request and events concept from
client and compositor side back and forth.
[For More detail precise Info read this](https://wayland.freedesktop.org/docs/html/ch04.html#sect-Protocol-Basic-Principles)

Code below to list out all global objects of the compositor

```c

#include "../protocol/xdg-shell-client-protocol.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <wayland-client.h>

struct app_state {

  // wl global objects
  struct wl_shm *shm;
  struct xdg_wm_base *xdg_wm_base;
  struct wl_compositor *comp;

  struct xdg_surface *xdg_surface;
  struct xdg_toplevel *xdg_toplevel;

  int32_t width;
  int32_t height;

  struct wl_surface *surface;
};

// Initialize the struct Instance
struct app_state app = {0};

//===============Setup Wayland Global Object Listerner===================//

void regis_list(void *data, struct wl_registry *wl_registry, uint32_t name,
                const char *interface, uint32_t version) {

  struct app_state *app = data;

  if (strcmp(interface, "wl_compositor") == 0) {
    app->comp =
        wl_registry_bind(wl_registry, name, &wl_compositor_interface, 6);
  }

  if (strcmp(interface, "wl_shm") == 0) {
    app->shm = wl_registry_bind(wl_registry, name, &wl_shm_interface, 1);
  }

  if (strcmp(interface, "xdg_wm_base") == 0) {
    app->xdg_wm_base =
        wl_registry_bind(wl_registry, name, &xdg_wm_base_interface, 1);
  }

  printf("interface: '%s', version: %u, name: %u\n", interface, version, name);
}

void regis_remove(void *data, struct wl_registry *wl_registry, uint32_t name) {
  (void)data;
  (void)(wl_registry);
  (void)name;
}

struct wl_registry_listener wl_registry_listener = {
    .global = regis_list,
    .global_remove = regis_remove,
};

int main() {

  struct wl_display *display = wl_display_connect(NULL);

  if (!display) {
    fprintf(stderr, "Failed to connect to Wayland display.\n");
    return 1;
  }

  struct wl_registry *regis = wl_display_get_registry(display);

  wl_registry_add_listener(regis, &wl_registry_listener, &app);

  wl_display_roundtrip(display);

  if (!app.comp || !app.xdg_wm_base || !app.shm) {
    fprintf(stderr, "missing globals\n");
    abort();
  }

  fprintf(stderr, "Connection established!\n");

  app.surface = wl_compositor_create_surface(app.comp);

}
```

For creating a simple window in wayland we need three objects

- wl_compositor (for creating **wl_surface**)
- xdg_wm_base (for window management)
- wl_shm (memory related operation)

The above code setup up the event handler callbacks in my case **regis_list** and **regis_remove**, they are grouped inside a single
listener struct.

`wl_registry_add_listener(regis, &wl_registry_listener, &app);` this function is where things add up and start making sense. So what happens
is this , whenever compositor events occur, Wayland dispatches them asynchronously and invoke these functions. **So Inside the listener
struct its the compositor which will call this functions which is listed not the clients.**

In the above add listener functions we pass wl registry struct instance which we got from calling **wl_display_get_registry** and second we
pass the struct user data for accessing our data throughout the callback functions , this way we don't have to use globals storing app
state.

## Setting Up Xdg Window

Now lets move on to the next setup setting up the Xdg Window.

As we have xdg_wm_base interface which is exposed as global clients,this interface allows clients to turn wl surface into windows in a
desktop environment. It defines the basic functionality needed for clients and the compositor to create windows that can be dragged,
resized, maximized, etc, as well as creating transient windows such as popup menus.

Below is the code to setup xdg-shell protocol callbacks:

```c
// xdg-shell protocol callbacks:
// - respond to xdg_wm_base ping
// - acknowledge xdg_surface configure events
// - handle xdg_toplevel configure/close events

// Xdg Base
void xdg_wm_base_ping(void *data, struct xdg_wm_base *xdg_wm_base,
                      uint32_t serial) {
  xdg_wm_base_pong(xdg_wm_base, serial);
};

struct xdg_wm_base_listener wm_base_listener = {
    .ping = xdg_wm_base_ping,
};

// Xdg Surface
void xdg_surface_handle_configure(void *data, struct xdg_surface *xdg_surface,
                                  uint32_t serial) {
  xdg_surface_ack_configure(xdg_surface, serial);
}

struct xdg_surface_listener xdg_surface_listener = {
    .configure = xdg_surface_handle_configure,
};

// Xdg Toplevel
void xdg_toplevel_configure(void *data, struct xdg_toplevel *xdg_toplevel,
                            int32_t width, int32_t height,
                            struct wl_array *states) {}

static void xdg_toplevel_handle_close(void *data,
                                      struct xdg_toplevel *xdg_toplevel) {}

struct xdg_toplevel_listener xdg_toplevel_listener = {
    .configure = xdg_toplevel_configure,
    .close = xdg_toplevel_handle_close,
};

```

Now I have setup the callback functions and struct listeners of xdg shell events for making a simple window.

Now lets create xdg surface and toplevel in main and also add listeners for them.

```c
    app.xdg_surface =
      xdg_wm_base_get_xdg_surface(app.xdg_wm_base, app.surface);

    app.xdg_toplevel = xdg_surface_get_toplevel(app.xdg_surface);

    /* Initial commit (required by protocol) */
    wl_surface_commit(surface);


  xdg_surface_add_listener(app.xdg_surface, &xdg_surface_listener, &app);

  xdg_toplevel_add_listener(app.xdg_toplevel, &xdg_toplevel_listener, &app);

  xdg_wm_base_add_listener(app.xdg_wm_base, &xdg_wm_base_listener, &app);
```

# Lets create pixel and put that into wl surface

I have almost setup most boilerplate code for making display a simple window in c , now lets add pixel in wl surface so we can actually see
the display.

Overall steps what will happen

- Allocate and fill a file-backed framebuffer in shared memory
- Register that memory with the compositor using wl_shm
- Create wl buffer , attach it to the surface
- Finally commit all the changes

Below is is the code

```c


  struct app_state *state = data;
  // Manually setting
  // int width = 800, height = 800;
  //
  int stride = state->width * 4;
  int size = stride * state->height;

  int fd = memfd_create("buf", 0);
  ftruncate(fd, size);

  uint32_t *pixels =
      mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

  //for (int i = 0; i < state->width * state->height; i++)
   // pixels[i] = 0xff2020ff;

    memset(pixels, 0xff, size);

  struct wl_shm_pool *pool = wl_shm_create_pool(app.shm, fd, size);
  struct wl_buffer *buffer = wl_shm_pool_create_buffer(
      pool, 0, app.width, app.height, stride, WL_SHM_FORMAT_ARGB8888);

  wl_surface_attach(state->surface, buffer, 0, 0);
  wl_surface_commit(state->surface);


```

The above mentioned code can be put inside xdg surface callback functin it works, also it can be put inside the while
(wl_display_dispatch(display) != -1) loop. This loop is important.

For window height and width to let compositor decide it xdg surface toplevel decides it.

```c

// Xdg Toplevel
void xdg_toplevel_configure(void *data, struct xdg_toplevel *xdg_toplevel,
                            int32_t width, int32_t height,
                            struct wl_array *states) {

  // if we dont manually decide the width and height let it decide we do it in
  // here in xdg toplevel callback function

  struct app_state *state = data;
  app.width = width;
  app.height = height;
}

```

**wl_display_roundtrip**: blocks until the compositor has processed previously sent requests and replied. It is commonly used after binding
globals to ensure required interfaces are available before continuing.

Here is the full code which is inside the main function.

```c
int main() {

  struct wl_display *display = wl_display_connect(NULL);

  if (!display) {
    fprintf(stderr, "Failed to connect to Wayland display.\n");
    return 1;
  }

  struct wl_registry *regis = wl_display_get_registry(display);

  wl_registry_add_listener(regis, &wl_registry_listener, &app);

  wl_display_roundtrip(display);

  if (!app.comp || !app.xdg_wm_base || !app.shm) {
    fprintf(stderr, "missing globals\n");
    abort();
  }

  fprintf(stderr, "Connection established!\n");

  app.surface = wl_compositor_create_surface(app.comp);

  app.xdg_surface = xdg_wm_base_get_xdg_surface(app.xdg_wm_base, app.surface);

  app.xdg_toplevel = xdg_surface_get_toplevel(app.xdg_surface);

  wl_surface_commit(app.surface); //This should be done once xdg surface has be assigned to wl surface , important step. For
                                  // For more info read the protocol doc

  xdg_surface_add_listener(app.xdg_surface, &xdg_surface_listener, &app);

  xdg_toplevel_add_listener(app.xdg_toplevel, &xdg_toplevel_listener, &app);

  xdg_wm_base_add_listener(app.xdg_wm_base, &xdg_wm_base_listener, &app);
  while (wl_display_dispatch(display) != -1) {
  }
}
```

**wl_surface_commit(app.surface)** The initial commit without a buffer is required to notify the compositor that the role (xdg_surface) has
been assigned. Only after this will the compositor send the first configure event.

**Important**: Since xdg shell protocol is not part of core wayland protocol , so we have to use the wayland scanner to generate .c and .h
code to use this protocol. Obviously you can chatgpt it how to do so , no point in mentioning the steps here.

**NOTE**: There are lots lots of issue with this code also i have ommited out a lots of details like how lifecycle works in wayland and
destroying the protocol objects once being used all those things but i have done it in my project wlok which you can checkout later. Also
wayland compositor is implemented differently in different environment depends on the the compositor itself how it implemented the protocol
I am running it on hyprland.

I will also provide the full code (repo) link and also my repo for a wayland locker which i am creating during which i came across wayland
and decided to write this blog.

Below is the demonstration.

<video width="640" height="360" controls>
  <source src=" https://res.cloudinary.com/dxistrivf/video/upload/v1772088408/wayland_compressed_z2xs8p.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

To change the window color to other you can uncomment this link i mentioned above and comment the line below.

```c
  for (int i = 0; i < state->width * state->height; i++)
   pixels[i] = 0xff2020ff;

  //  memset(pixels, 0xff, size);

```

There are lots of details i didnt mentioned about how protocols work , all those i will put the reference to site below.

- [My Repo for all the code above](https://codeberg.org/vishalkrkamat/simple_wayland_client_from_scratch)
- [Wlok repo](https://github.com/vishalkrkamat/wlok)[Still in making]

## 📚 Wayland References

Here are some useful resources for learning and exploring Wayland:

- [Wayland Protocols Documentation](https://wayland.app/protocols/)
- [Writing Wayland Clients (by bugaevc)](https://bugaevc.gitbooks.io/writing-wayland-clients/content/)
- [Official Wayland Documentation](https://wayland.freedesktop.org/docs/html/index.html)
- [The Wayland Book – Introduction](https://wayland-book.com/introduction.html)
- [nots1dd learning about wayland](https://github.com/nots1dd/mywayland?tab=readme-ov-file)
