# photoshop-cs3-bonjour-error-1920-fix
Photoshop CS3 crashed → reinstall broke it worse → installer blamed a service called "Bonjour" (yes, that Bonjour) → turns out my Nikon camera's wireless transfer utility was squatting on the same mDNS port the whole time. Full diagnostic trail from crash dump to SFC/DISM to the actual culprit: a 2013 Nikon tool nobody invited.
