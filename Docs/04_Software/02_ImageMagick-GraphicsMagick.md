ImageMagick / GraphicsMagic
===========================

Depending on your preference, install ImageMagick

```sh
emerge imagemagick
```

or GraphicsMagick

```sh
emerge graphicsmagick
```

Japanese fonts
--------------

> **ATTENTION**: The following information doesn't seem to be accurate anymore with current versions.

For being able to process PDF files with Japanese fonts, some additional requirements need to be met. In your `/etc/portage/make.conf`, add:
 
```sh
LINGUAS="de ja"
```

Recompile Ghostscript:

```sh
emerge -av ghostscript-gpl
```

Install the Japanese fonts

* MS Mincho
* MS Gothic

by downloading them from somewhere and extracting the TrueType files into the directory `/usr/share/fonts/ms-japanese-truetype` (you will need to create this). Then alter the Ghostscript configuration `/usr/share/ghostscript/8.71/Resource/Init/cidfmap.ja` to use these fonts instead of `Adobe-Japan1` and `Adobe-Japan2`:

```sh
<code>/MSGothic  << /FileType /TrueType /Path  (/usr/share/fonts/</code><code>ms-japanese-truetype</code><code>/MSGOTHIC.TTF) /CSI [(Japan1)  2] >> ;
/MSMincho << /FileType /TrueType /Path  (/usr/share/fonts/ms-japanese-truetype/MSMINCHO.TTF) /CSI [(Japan1)  2] >> ;

/Adobe-Japan1           /MSGothic ;
/Adobe-Japan2           /MSGothic ;</code>
```

Finally remove the flag `-dSafer` from all PDF and PS delegate configurations in `/usr/lib/ImageMagick-x.x.x/config/delegates.xml`.

___
[Back to Overview](01_Overview.md)