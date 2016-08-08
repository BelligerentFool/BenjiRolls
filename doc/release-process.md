Release Process
====================

* * *

###update (commit) version in sources


	benjirolls-qt.pro
	contrib/verifysfbinaries/verify.sh
	doc/README*
	share/setup.nsi
	src/clientversion.h (change CLIENT_VERSION_IS_RELEASE to true)

###tag version in git

	git tag -a v0.8.0

###write release notes. git shortlog helps a lot, for example:

	git shortlog --no-merges v0.7.2..v0.8.0

* * *

##perform gitian builds

 From a directory containing the benjirolls source, gitian-builder and gitian.sigs
  
	export SIGNER=(your gitian key, ie bluematt, sipa, etc)
	export VERSION=0.8.0
	cd ./gitian-builder

 Fetch and build inputs: (first time, or when dependency versions change)

	mkdir -p inputs; cd inputs/
	wget 'http://miniupnp.free.fr/files/download.php?file=miniupnpc-1.6.tar.gz' -O miniupnpc-1.6.tar.gz
	wget 'http://www.openssl.org/source/openssl-1.0.1c.tar.gz'
	wget 'http://download.oracle.com/berkeley-db/db-4.8.30.NC.tar.gz'
	wget 'http://zlib.net/zlib-1.2.6.tar.gz'
	wget 'ftp://ftp.simplesystems.org/pub/libpng/png/src/libpng-1.5.9.tar.gz'
	wget 'http://fukuchi.org/works/qrencode/qrencode-3.2.0.tar.bz2'
	wget 'http://downloads.sourceforge.net/project/boost/boost/1.50.0/boost_1_50_0.tar.bz2'
	wget 'http://releases.qt-project.org/qt4/source/qt-everywhere-opensource-src-4.8.3.tar.gz'
	cd ..
	./bin/gbuild ../benjirolls/contrib/gitian-descriptors/boost-win32.yml
	mv build/out/boost-win32-1.50.0-gitian2.zip inputs/
	./bin/gbuild ../benjirolls/contrib/gitian-descriptors/qt-win32.yml
	mv build/out/qt-win32-4.8.3-gitian-r1.zip inputs/
	./bin/gbuild ../benjirolls/contrib/gitian-descriptors/deps-win32.yml
	mv build/out/benjirolls-deps-0.0.5.zip inputs/

 Build benjirollsd and benjirolls-qt on Linux32, Linux64, and Win32:
  
	./bin/gbuild --commit benjirolls=v${VERSION} ../benjirolls/contrib/gitian-descriptors/gitian.yml
	./bin/gsign --signer $SIGNER --release ${VERSION} --destination ../gitian.sigs/ ../benjirolls/contrib/gitian-descriptors/gitian.yml
	pushd build/out
	zip -r benjirolls-${VERSION}-linux-gitian.zip *
	mv benjirolls-${VERSION}-linux-gitian.zip ../../
	popd
	./bin/gbuild --commit benjirolls=v${VERSION} ../benjirolls/contrib/gitian-descriptors/gitian-win32.yml
	./bin/gsign --signer $SIGNER --release ${VERSION}-win32 --destination ../gitian.sigs/ ../benjirolls/contrib/gitian-descriptors/gitian-win32.yml
	pushd build/out
	zip -r benjirolls-${VERSION}-win32-gitian.zip *
	mv benjirolls-${VERSION}-win32-gitian.zip ../../
	popd

  Build output expected:

  1. linux 32-bit and 64-bit binaries + source (benjirolls-${VERSION}-linux-gitian.zip)
  2. windows 32-bit binary, installer + source (benjirolls-${VERSION}-win32-gitian.zip)
  3. Gitian signatures (in gitian.sigs/${VERSION}[-win32]/(your gitian key)/

repackage gitian builds for release as stand-alone zip/tar/installer exe

**Linux .tar.gz:**

	unzip benjirolls-${VERSION}-linux-gitian.zip -d benjirolls-${VERSION}-linux
	tar czvf benjirolls-${VERSION}-linux.tar.gz benjirolls-${VERSION}-linux
	rm -rf benjirolls-${VERSION}-linux

**Windows .zip and setup.exe:**

	unzip benjirolls-${VERSION}-win32-gitian.zip -d benjirolls-${VERSION}-win32
	mv benjirolls-${VERSION}-win32/benjirolls-*-setup.exe .
	zip -r benjirolls-${VERSION}-win32.zip bitcoin-${VERSION}-win32
	rm -rf benjirolls-${VERSION}-win32

**Perform Mac build:**

  OSX binaries are created by Gavin Andresen on a 32-bit, OSX 10.6 machine.

	qmake RELEASE=1 USE_UPNP=1 USE_QRCODE=1 benjirolls-qt.pro
	make
	export QTDIR=/opt/local/share/qt4  # needed to find translations/qt_*.qm files
	T=$(contrib/qt_translations.py $QTDIR/translations src/qt/locale)
	python2.7 share/qt/clean_mac_info_plist.py
	python2.7 contrib/macdeploy/macdeployqtplus Bitcoin-Qt.app -add-qt-tr $T -dmg -fancy contrib/macdeploy/fancy.plist

 Build output expected: Bitcoin-Qt.dmg

###Next steps:

* Code-sign Windows -setup.exe (in a Windows virtual machine) and
  OSX Bitcoin-Qt.app (Note: only Gavin has the code-signing keys currently)

* upload builds to SourceForge

* create SHA256SUMS for builds, and PGP-sign it

* update benjirolls.org version
  make sure all OS download links go to the right versions

* update forum version

* update wiki download links

* update wiki changelog: [https://en.benjirolls.it/wiki/Changelog](https://en.bitcoin.it/wiki/Changelog)

Commit your signature to gitian.sigs:

	pushd gitian.sigs
	git add ${VERSION}/${SIGNER}
	git add ${VERSION}-win32/${SIGNER}
	git commit -a
	git push  # Assuming you can push to the gitian.sigs tree
	popd

-------------------------------------------------------------------------

### After 3 or more people have gitian-built, repackage gitian-signed zips:

From a directory containing benjirolls source, gitian.sigs and gitian zips

	export VERSION=0.5.1
	mkdir benjirolls-${VERSION}-linux-gitian
	pushd benjirolls-${VERSION}-linux-gitian
	unzip ../benjirolls-${VERSION}-linux-gitian.zip
	mkdir gitian
	cp ../benjirolls/contrib/gitian-downloader/*.pgp ./gitian/
	for signer in $(ls ../gitian.sigs/${VERSION}/); do
	 cp ../gitian.sigs/${VERSION}/${signer}/benjirolls-build.assert ./gitian/${signer}-build.assert
	 cp ../gitian.sigs/${VERSION}/${signer}/benjirolls-build.assert.sig ./gitian/${signer}-build.assert.sig
	done
	zip -r benjirolls-${VERSION}-linux-gitian.zip *
	cp benjirolls-${VERSION}-linux-gitian.zip ../
	popd
	mkdir benjirolls-${VERSION}-win32-gitian
	pushd benjirolls-${VERSION}-win32-gitian
	unzip ../benjirolls-${VERSION}-win32-gitian.zip
	mkdir gitian
	cp ../benjirolls/contrib/gitian-downloader/*.pgp ./gitian/
	for signer in $(ls ../gitian.sigs/${VERSION}-win32/); do
	 cp ../gitian.sigs/${VERSION}-win32/${signer}/benjirolls-build.assert ./gitian/${signer}-build.assert
	 cp ../gitian.sigs/${VERSION}-win32/${signer}/benjirolls-build.assert.sig ./gitian/${signer}-build.assert.sig
	done
	zip -r benjirolls-${VERSION}-win32-gitian.zip *
	cp benjirolls-${VERSION}-win32-gitian.zip ../
	popd

- Upload gitian zips to SourceForge
- Celebrate 
