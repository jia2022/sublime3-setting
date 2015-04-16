# sublime3-setting
sublime3-setting文件包。


保险起见，对于新手适用于新安装的系统，要是做了很多操作，那就不好说了

1.安装拼音fcitx这个字体引擎.

    国内一般换成163的源，更新，执行:
    sudo install fcitx fcitx-pinyin 
    到http://pinyin.sogou.com/linux/?r=pinyin下载deb包安装
    language support里输入方式选为fcitx
    重启，sougoupinyin 安装成功

2.安装sublime text 3

    sudo add-apt-repository ppa:webupd8team/sublime-text-3
    sudo apt-get update
    sudo apt-get install sublime-text-installer

3.解决中文输入问题

    cd /opt/sublime_text
    sudo touch sublime_imfix.c
    sudo gedit sublime_imfix.c

4.然后把以下内容粘贴进去并且保存

    /*
    sublime-imfix.c
    Use LD_PRELOAD to interpose some function to fix sublime input method support for linux.
    By Cjacker Huang <jianzhong.huang at i-soft.com.cn>
    gcc -shared -o libsublime-imfix.so sublime_imfix.c  `pkg-config --libs --cflags gtk+-2.0` -fPIC
    LD_PRELOAD=./libsublime-imfix.so sublime_text
    */
    #include <gtk/gtk.h>
    #include <gdk/gdkx.h>
    typedef GdkSegment GdkRegionBox;
    struct _GdkRegion
    {
      long size;
      long numRects;
      GdkRegionBox *rects;
      GdkRegionBox extents;
    };
    GtkIMContext *local_context;
    void
    gdk_region_get_clipbox (const GdkRegion *region,
                GdkRectangle    *rectangle)
    {
      g_return_if_fail (region != NULL);
      g_return_if_fail (rectangle != NULL);
      
      rectangle->x = region->extents.x1;
      rectangle->y = region->extents.y1;
      rectangle->width = region->extents.x2 - region->extents.x1;
      rectangle->height = region->extents.y2 - region->extents.y1;
      GdkRectangle rect;
      rect.x = rectangle->x;
      rect.y = rectangle->y;
      rect.width = 0;
      rect.height = rectangle->height; 
      //The caret width is 2; 
      //Maybe sometimes we will make a mistake, but for most of the time, it should be the caret.
      if(rectangle->width == 2 && GTK_IS_IM_CONTEXT(local_context)) {
            gtk_im_context_set_cursor_location(local_context, rectangle);
      }
    }
    //this is needed, for example, if you input something in file dialog and return back the edit area
    //context will lost, so here we set it again.
    static GdkFilterReturn event_filter (GdkXEvent *xevent, GdkEvent *event, gpointer im_context)
    {
        XEvent *xev = (XEvent *)xevent;
        if(xev->type == KeyRelease && GTK_IS_IM_CONTEXT(im_context)) {
           GdkWindow * win = g_object_get_data(G_OBJECT(im_context),"window");
           if(GDK_IS_WINDOW(win))
             gtk_im_context_set_client_window(im_context, win);
        }
        return GDK_FILTER_CONTINUE;
    }


    void gtk_im_context_set_client_window (GtkIMContext *context,
              GdkWindow    *window)
    {
      GtkIMContextClass *klass;
      g_return_if_fail (GTK_IS_IM_CONTEXT (context));
      klass = GTK_IM_CONTEXT_GET_CLASS (context);
      if (klass->set_client_window)
        klass->set_client_window (context, window);
        
      if(!GDK_IS_WINDOW (window))
        return;
      g_object_set_data(G_OBJECT(context),"window",window);
      int width = gdk_window_get_width(window);
      int height = gdk_window_get_height(window);
      if(width != 0 && height !=0) {
        gtk_im_context_focus_in(context);
        local_context = context;
      }
      gdk_window_add_filter (window, event_filter, context); 
    }

5.再执行：

    sudo apt-get install build-essential libgtk2.0-dev
    sudo gcc -shared -o libsublime-imfix.so sublime_imfix.c  `pkg-config --libs --cflags gtk+-2.0` -fPIC

6.再编辑sublime快捷方式

    sudo gedit /usr/share/applications/sublime-text.desktop

7.把以下内容替换原来内容并且保存

    [Desktop Entry]
    Version=1.0
    Type=Application
    Name=Sublime Text
    GenericName=Text Editor
    Comment=Sophisticated text editor for code, markup and prose
    Exec=bash -c 'LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so /opt/sublime_text/sublime_text' %F
    Terminal=false
    MimeType=text/plain;
    Icon=sublime-text
    Categories=TextEditor;Development;Utility;
    StartupNotify=true
    Actions=Window;Document;
    X-Desktop-File-Install-Version=0.22
    [Desktop Action Window]
    Name=New Window
    Exec=bash -c 'LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so /opt/sublime_text/sublime_text' -n
    OnlyShowIn=Unity;
    [Desktop Action Document]
    Name=New File
    Exec=bash -c 'LD_PRELOAD=/opt/sublime_text/libsublime-imfix.so /opt/sublime_text/sublime_text' --command new_file
    OnlyShowIn=Unity;
    
    

到此完工，可以正常输入中文。

8.可选，使用sublime配置文件

务必至少打开一次sublime text 3, 并且在关闭状态下执行以下操作

    cd ~/.config/sublime-text-3
    rm -rf Installed\ Packages
    rm Packages/User/Package\ Control.sublime-settings
    rm Packages/User/Preferences.sublime-settings
    git clone https://github.com/jia2022/sublime3-setting.git
    cp -R sublime3-setting/* .
