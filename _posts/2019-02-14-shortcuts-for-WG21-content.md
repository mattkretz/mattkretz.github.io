---
layout: post
title: "Shortcuts for WG21 content"
tags: [C++, WG21, KDE]
author: Matthias Kretz
---

## KDE Web Shortcuts for C++

Since at least KDE 2, the runner (`krunner` - per default on `Alt-F2`) supports 
configurable prefixes that resolve to a URL that is then opened like 
`kioclient5 exec` would open them. I created the following shortcuts:

- [wg21:](/assets/wg21.desktop) Resolves to https://wg21.link/ e.g. for easy 
retrieval of papers by paper number, issues, or links to the working draft at a 
given stable name.
- [wg21d:](/assets/wg21d.desktop) Searches through the C++ working **d**raft.
- [wg21p:](/assets/wg21p.desktop) Searches through all committee **p**apers.
- [wg21w:](/assets/wg21w.desktop) Searches through all **w**ikis.
- [wg21i:](/assets/wg21i.desktop) Opens the working draft **i**ndex at the 
given anchor.
- [@cwg:](/assets/@cwg.desktop) Searches through the CWG reflector archive.
- [@ewg:](/assets/@ewg.desktop) Searches through the EWG reflector archive.
- [@lwg:](/assets/@lwg.desktop) Searches through the LWG reflector archive.
- [@lewg:](/assets/@lewg.desktop) Searches through the LEWG reflector archive.
- [@sg1:](/assets/@sg1.desktop) Searches through the SG1 reflector archive.
- [cpp:](/assets/cpp.desktop) Searches on cppreference.com.

Save these files ([tarball of all files](/assets/web_shortcuts.tar.gz)) to 
`~/.local/share/kservices5/searchproviders/` to use them yourself. (A restart 
of `krunner` is needed it seems.)
