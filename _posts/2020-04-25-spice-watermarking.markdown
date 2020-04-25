---
layout: post
title:  "SPICE watermarking"
date:   2020-04-25 10:59:00 +0100
categories: spice
---

Implementing watermarking on SPICE.

I saw an article implementing watermarking in Citrix products.
Basically all screens with a semitransparent image overlay on them to be able to track the original screenshots.

How could this be implemented in SPICE ?

After some thought some ways to implement it came into my mind
1. *spice-streaming-agent plugin*. It would be easy to implement using
   Gstreamer plugin however it won't help much as it would address
only a small portions of cases;
2. *watermarking just before compressing images*. It could be done
   however there are some issues. GLZ require the images to not be
changed so we would have to do a copy before applying the overlay.
If the drawing operation is not opaque we would need flat the image
before overlaying. Operations like moving windows would require to
convert image copies to opaque drawing overlaying and sending
continuously reducing the efficiency of these operation. This would
also load more the server with overlaying, drawing and other
operations. I would also expect compression to be less efficient.
3. *watermarking on the client side*. SPICE protocol was designed, at
   least initially to be able to send 2D graphic commands to the
client. Why not using this capability? Also the protocol allows to use
additional not visible (offline) surfaces. The network usage, beside
sending the overlay image and some additional commands, should not
change that much. I decided to try to implement (as a PoC) this
method.

So, the roughly idea is to redirect the commands to an offline surface
created on purpose and then use this surface as source for the overlay.

I briefly checked the client (spice-gtk) sources if it seems doable and
it was. Also checked `spice.proto` file (spice-common) to see the amount
of commands/structure to patch in order to redirect the commands.

As it's good to see some partial results during a PoC to see if we are
going in the right direction I tried to define some steps and tests:
1. *create/destroy surface*. When a primary (0) surface is create this
   should also create an offline surface. The client should not complain
about but nothing else visible;
2. *all commands redirected to offline surface*. If we redirect all
   commands correctly on the client we should not see errors and the
visible surface should not change (that is should stay black);
3. *copy to visible surface*. In order to check that above stages is
  good to copy the modified part of the changed surface to the visible one.
  Now we should be able to interact with the VM as nothing was changed.
4. *some real overlay*. Just try to do something more complicated than
  just copy the surface.


## create/destroy surface

Looking at `spice.proto` the commands are `surface_create` and `surface_destroy`, so
the messages to look in the code are `SPICE_MSG_DISPLAY_SURFACE_CREATE` and
`SPICE_MSG_DISPLAY_SURFACE_DESTROY`. From these following the caller you can find
code in `dcc-send.c`, `dcc.c` and the `StreamChannel`. Ignoring for the PoC the
`StreamChannel`. The patch was pretty easy. Note that I used for the surface number
a surface over the server limit, this is not a problem for the client.

{% highlight diff %}
diff --git a/server/dcc.c b/server/dcc.c
index 6d16da0ec..1375509d5 100644
--- a/server/dcc.c
+++ b/server/dcc.c
@@ -30,6 +30,8 @@ G_DEFINE_TYPE(DisplayChannelClient, display_channel_client, TYPE_COMMON_GRAPHICS
 #define DISPLAY_CLIENT_SHORT_TIMEOUT 15000000000ULL //nano
 #define DISPLAY_FREE_LIST_DEFAULT_SIZE 128
 
+#define fake_surface_id NUM_SURFACES
+
 enum
 {
     PROP0,
@@ -311,6 +313,13 @@ void dcc_create_surface(DisplayChannelClient *dcc, int surface_id)
                                          surface->context.format, flags);
     dcc->priv->surface_client_created[surface_id] = TRUE;
     red_channel_client_pipe_add(RED_CHANNEL_CLIENT(dcc), &create->base);
+    if (surface_id == 0) {
+        create = red_surface_create_item_new(RED_CHANNEL(display),
+                                             fake_surface_id, surface->context.width,
+                                             surface->context.height,
+                                             surface->context.format, 0);
+        red_channel_client_pipe_add(RED_CHANNEL_CLIENT(dcc), &create->base);
+    }
 }
 
 // adding the pipe item after pos. If pos == NULL, adding to head.
@@ -745,6 +754,10 @@ void dcc_destroy_surface(DisplayChannelClient *dcc, uint32_t surface_id)
     dcc->priv->surface_client_created[surface_id] = FALSE;
     destroy = red_surface_destroy_item_new(surface_id);
     red_channel_client_pipe_add(RED_CHANNEL_CLIENT(dcc), &destroy->base);
+    if (surface_id == 0) {
+        destroy = red_surface_destroy_item_new(fake_surface_id);
+        red_channel_client_pipe_add(RED_CHANNEL_CLIENT(dcc), &destroy->base);
+    }
 }
 
 #define MIN_DIMENSION_TO_QUIC 3
{% endhighlight %}

Well, client did not complain.


## all commands redirected to offline surface

Looking at `spice.proto`, specifically for `surface_id` you can see that lot
of commands contains a `DisplayBase` structure with a `surface_id` field and
also `Image` of `SURFACE` type can contain this field. For the PoC I decided
to ignore `stream_create` commands and `StreamChannel` (not that would be
extremely difficult).

Well, patch was surprisingly small!

{% highlight diff %}
diff --git a/server/dcc-send.c b/server/dcc-send.c
index 909c237ba..0cbc5a8ec 100644
--- a/server/dcc-send.c
+++ b/server/dcc-send.c
@@ -47,6 +47,14 @@ typedef struct BitmapData {
     SpiceRect lossy_rect;
 } BitmapData;
 
+#define fake_surface_id NUM_SURFACES
+
+static inline
+uint32_t fix_fake(uint32_t surface_id)
+{
+    return surface_id == 0 ? fake_surface_id : surface_id;
+}
+
 static int dcc_pixmap_cache_unlocked_hit(DisplayChannelClient *dcc, uint64_t id, int *lossy)
 {
     PixmapCache *cache = dcc->priv->pixmap_cache;
@@ -315,7 +323,7 @@ static void fill_base(SpiceMarshaller *base_marshaller, Drawable *drawable)
 {
     SpiceMsgDisplayBase base;
 
-    base.surface_id = drawable->surface_id;
+    base.surface_id = fix_fake(drawable->surface_id);
     base.box = drawable->red_drawable->bbox;
     base.clip = drawable->red_drawable->clip;
 
@@ -421,7 +429,7 @@ static FillBitsType fill_bits(DisplayChannelClient *dcc, SpiceMarshaller *m,
         image.descriptor.width = surface->context.width;
         image.descriptor.height = surface->context.height;
 
-        image.u.surface.surface_id = surface_id;
+        image.u.surface.surface_id = fix_fake(surface_id);
         spice_marshall_Image(m, &image,
                              &bitmap_palette_out, &lzplt_palette_out);
         spice_assert(bitmap_palette_out == NULL);
@@ -1963,7 +1971,7 @@ static void red_marshall_image(RedChannelClient *rcc,
 
     red_channel_client_init_send_data(rcc, SPICE_MSG_DISPLAY_DRAW_COPY);
 
-    copy.base.surface_id = item->surface_id;
+    copy.base.surface_id = fix_fake(item->surface_id);
     copy.base.box.left = item->pos.x;
     copy.base.box.top = item->pos.y;
     copy.base.box.right = item->pos.x + bitmap.x;
@@ -2166,7 +2174,7 @@ static void marshall_stream_start(RedChannelClient *rcc,
     SpiceMsgDisplayStreamCreate stream_create;
     SpiceClipRects clip_rects;
 
-    stream_create.surface_id = 0;
+    stream_create.surface_id = fix_fake(0);
     stream_create.id = display_channel_get_video_stream_id(DCC_TO_DC(dcc), stream);
     stream_create.flags = stream->top_down ? SPICE_STREAM_FLAGS_TOP_DOWN : 0;
     stream_create.codec_type = agent->video_encoder->codec_type;
@@ -2240,7 +2248,7 @@ static void marshall_upgrade(RedChannelClient *rcc, SpiceMarshaller *m,
     spice_assert(red_drawable->u.copy.rop_descriptor == SPICE_ROPD_OP_PUT);
     spice_assert(red_drawable->u.copy.mask.bitmap == 0);
 
-    copy.base.surface_id = 0;
+    copy.base.surface_id = fix_fake(0);
     copy.base.box = red_drawable->bbox;
     copy.base.clip.type = SPICE_CLIP_TYPE_RECTS;
     copy.base.clip.rects = item->rects;
{% endhighlight %}

And client is showing a black display, as expected.


## copy to visible surface

The idea here is to add a copy command when we see a drawing command.
As each command contains the box we don't need much to analyze the command.
The message for the copy is `SPICE_MSG_DISPLAY_DRAW_COPY` so come copy and
paste from the code that send this message helps.
As mostly of these commands call `fill_base` to do their job changing this
function helps reducing the changes for the PoC. Also using some `#define`
trick does not hurt.
I added an item to `DisplayChannel` for these new messages.

{% highlight diff %}
diff --git a/server/dcc-send.c b/server/dcc-send.c
index 0cbc5a8ec..512c2dc85 100644
--- a/server/dcc-send.c
+++ b/server/dcc-send.c
@@ -319,7 +319,68 @@ static void send_free_list(RedChannelClient *rcc)
     spice_marshaller_add_uint32(urgent_marshaller, inval_offset + sub_arr_offset);
 }
 
-static void fill_base(SpiceMarshaller *base_marshaller, Drawable *drawable)
+typedef struct RedFakeCopyItem {
+    RedPipeItem base;
+    SpiceRect box;
+} RedFakeCopyItem;
+
+static void
+copy_base(RedChannelClient *rcc, const SpiceMsgDisplayBase *base)
+{
+    if (base->surface_id != fake_surface_id) {
+        return;
+    }
+
+    RedFakeCopyItem *item = g_new(RedFakeCopyItem, 1);
+    red_pipe_item_init(&item->base, RED_PIPE_ITEM_TYPE_FAKE_COPY);
+    item->box = base->box;
+    red_channel_client_pipe_add_tail(rcc, &item->base);
+}
+
+static void
+marshall_fake_copy(RedChannelClient *rcc, SpiceMarshaller *m, RedFakeCopyItem *item)
+{
+    // copy from fake surface to primary surface
+    DisplayChannelClient *dcc = DISPLAY_CHANNEL_CLIENT(rcc);
+    SpiceMsgDisplayDrawCopy copy;
+    SpiceImage image;
+    SpiceMarshaller *src_bitmap_out, *mask_bitmap_out;
+    SpiceMarshaller *bitmap_palette_out, *lzplt_palette_out;
+    RedSurface *surface = &DCC_TO_DC(dcc)->priv->surfaces[0];
+
+    if (!surface->context.canvas) {
+        return;
+    }
+
+    red_channel_client_init_send_data(rcc, SPICE_MSG_DISPLAY_DRAW_COPY);
+
+    image.descriptor.id = 0;
+    image.descriptor.type = SPICE_IMAGE_TYPE_SURFACE;
+    image.descriptor.flags = 0;
+    image.descriptor.width = surface->context.width;
+    image.descriptor.height = surface->context.height;
+    image.u.surface.surface_id = fake_surface_id;
+
+    copy.base.surface_id = 0;
+    copy.base.box = item->box;
+    copy.base.clip.type = SPICE_CLIP_TYPE_NONE;
+    copy.base.clip.rects = NULL;
+    copy.data.src_bitmap = &image;
+    copy.data.src_area = item->box;
+    copy.data.rop_descriptor = SPICE_ROPD_OP_PUT;
+    copy.data.scale_mode = 0;
+    copy.data.mask.flags = 0;
+    copy.data.mask.pos.x = 0;
+    copy.data.mask.pos.y = 0;
+    copy.data.mask.bitmap = NULL;
+
+    spice_marshall_msg_display_draw_copy(m, &copy,
+                                         &src_bitmap_out, &mask_bitmap_out);
+
+    spice_marshall_Image(src_bitmap_out, &image, &bitmap_palette_out, &lzplt_palette_out);
+}
+
+static void fill_base(RedChannelClient *rcc, SpiceMarshaller *base_marshaller, Drawable *drawable)
 {
     SpiceMsgDisplayBase base;
 
@@ -328,7 +389,10 @@ static void fill_base(SpiceMarshaller *base_marshaller, Drawable *drawable)
     base.clip = drawable->red_drawable->clip;
 
     spice_marshall_DisplayBase(base_marshaller, &base);
+
+    copy_base(rcc, &base);
 }
+#define fill_base(a,b) fill_base(rcc,a,b)
 
 static void marshaller_compress_buf_free(uint8_t *data, void *opaque)
 {
@@ -1989,6 +2053,8 @@ static void red_marshall_image(RedChannelClient *rcc,
     copy.data.mask.pos.y = 0;
     copy.data.mask.bitmap = 0;
 
+    copy_base(rcc, &copy.base);
+
     spice_marshall_msg_display_draw_copy(m, &copy,
                                          &src_bitmap_out, &mask_bitmap_out);
 
@@ -2254,6 +2320,8 @@ static void marshall_upgrade(RedChannelClient *rcc, SpiceMarshaller *m,
     copy.base.clip.rects = item->rects;
     copy.data = red_drawable->u.copy;
 
+    copy_base(rcc, &copy.base);
+
     spice_marshall_msg_display_draw_copy(m, &copy,
                                          &src_bitmap_out, &mask_bitmap_out);
 
@@ -2463,6 +2531,11 @@ void dcc_send_item(RedChannelClient *rcc, RedPipeItem *pipe_item)
     case RED_PIPE_ITEM_TYPE_GL_DRAW:
         marshall_gl_draw(rcc, m, pipe_item);
         break;
+    case RED_PIPE_ITEM_TYPE_FAKE_COPY: {
+        RedFakeCopyItem *item = SPICE_UPCAST(RedFakeCopyItem, pipe_item);
+        marshall_fake_copy(rcc, m, item);
+        break;
+    }
     default:
         spice_warn_if_reached();
     }
diff --git a/server/display-channel-private.h b/server/display-channel-private.h
index 4cdae8dcb..633d6d913 100644
--- a/server/display-channel-private.h
+++ b/server/display-channel-private.h
@@ -159,6 +159,7 @@ enum {
     RED_PIPE_ITEM_TYPE_STREAM_ACTIVATE_REPORT,
     RED_PIPE_ITEM_TYPE_GL_SCANOUT,
     RED_PIPE_ITEM_TYPE_GL_DRAW,
+    RED_PIPE_ITEM_TYPE_FAKE_COPY,
 };
 
 void drawable_unref(Drawable *drawable);
{% endhighlight %}

Well, client is still happy and we seem to interact with the VM without these patches
were here! It took a bit of time for this last patch (mostly some silly mistakes) but
was not so complex as expected.


## some "real" overlay

Well, honestly the PoC is mostly in place. The full overlay would require to send the
image, store somewhere (a new offline surface or trying to use image cache, possibly
easier the first), how to do the overlay (use `draw_alpha_blend` commands maybe on
an additional offline surface for double buffering to avoid possible flickering).

So, to finish the PoC I decided instead to go for something much easier; just
invert the image surface!

{% highlight diff %}
diff --git a/server/dcc-send.c b/server/dcc-send.c
index 512c2dc85..0b200b46e 100644
--- a/server/dcc-send.c
+++ b/server/dcc-send.c
@@ -367,7 +367,7 @@ marshall_fake_copy(RedChannelClient *rcc, SpiceMarshaller *m, RedFakeCopyItem *i
     copy.base.clip.rects = NULL;
     copy.data.src_bitmap = &image;
     copy.data.src_area = item->box;
-    copy.data.rop_descriptor = SPICE_ROPD_OP_PUT;
+    copy.data.rop_descriptor = SPICE_ROPD_OP_PUT|SPICE_ROPD_INVERS_RES;
     copy.data.scale_mode = 0;
     copy.data.mask.flags = 0;
     copy.data.mask.pos.x = 0;
{% endhighlight %}

Well, it worked! Only a small issue with text mode due to initial screen black
and not inverted but I left as exercise if somebody wants to fix it.


## more than PoC

Not sure. But surely it would require to finish the small stuff (`StreamChannel`, normal streams),
have a way to configure the overlay image and have the code conditional to enabling the
watermarking.
