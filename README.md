# Tollwerk setup for Gentoo boxes

This documentation walks you through the steps we usually take for setting up a new Gentoo Linux server. All of our servers use a hardened AMD64 architecture. Setting up a Gentoo box is not exactly trivial and requires a lot of manual work — the documentation requires you to have a certain knowledge about Linux, how to edit files with `nano` and so on.

1. [Live CD](Docs/01_Live-CD.md)
2. [Hard Drives](Docs/02_Hard-Drives.md)
3. System Installation
    1. [System Sources](Docs/03_Installation/01_System-Sources.md)
    2. [Kernel Setup](Docs/03_Installation/02_Kernel.md)
    3. [Basic Configuration](Docs/03_Installation/03_Basic-Configuration.md)
    4. [Basic Software](Docs/03_Installation/04_Basic-Software.md)
    5. [Finish Setup](Docs/03_Installation/05_Finish-Setup.md)
4. [Software Installation](Docs/04_Software/01_Overview.md)
    1. [ImageMagick / GraphicsMagick](Docs/04_Software/02_ImageMagick-GraphicsMagick.md)
    2. [MySQL](Docs/04_Software/03_MySQL.md)
    3. [OpenSSL](Docs/04_Software/04_OpenSSL.md)
    4. [Apache / PHP](Docs/04_Software/05_Apache-PHP.md) with Event MPM, HTTP2, PHP-FPM and PHP 7
    5. [Let's Encrypt](Docs/04_Software/06_Letsencrypt.md)
5. Shared hosting
    1. [Webhosting Setup](Docs/05_Shared_Hosting/01_Webhosting.md)