# ROCK Pi Quad SATA

Top Board control program for ROCK Pi Quad SATA

[Quad SATA HAT wiki](<https://wiki.radxa.com/Dual_Quad_SATA_HAT>)

# How to use

Compile the code into deb package
```
$ cd rockpi-quad
$ chmod 0755 rockpi-quad/DEBIAN/postinst
$ chmod 0755 rockpi-quad/DEBIAN/prerm
$ dpkg-deb --build . rockpi-quad.custom.deb
```
Install the deb package
```
$ sudo dpkg -i rockpi-quad.custom.deb
```
