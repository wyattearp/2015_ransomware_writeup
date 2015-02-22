## Weekend Ransomware Analysis
One of my wife's co-workers managed to catch a crap piece of ransomware this last week; I believe that she got it from an email that looked similar to this:

> Our company’s courier couldn’t make the delivery of package.
>
>     REASON: Postal code contains an error.
>     DELIVERY STATUS: Sort Order
>     SERVICE: One-day Shipping
>     NUMBER OF YOUR PARCEL: USPS00272452
>     FEATURES: No
>
> Label is enclosed to the letter in .doc format.
> Print a label and show it at your post office.
>
> An additional information:
>
> You can find the information about the procedure and conditions of parcels keeping in the nearest office.
>
> Thank you for using our services.
> USPS Global.
>
> *** This is an automatically generated email, please do not reply ***
>
> CONFIDENTIALITY NOTICE:
> This electronic mail transmission and any attached files contain information intended for the exclusive use of the individual or entity to whom it is addressed and may contain information belonging to the sender (USPS , Inc.) that is proprietary, privileged, confidential and/or protected from disclosure under applicable law. If you are not the intended recipient, you are hereby notified that any viewing, copying, disclosure or distributions of this electronic message are violations of federal law. Please notify the sender of any unintended recipients and delete the original message without making any copies.  Thank You

I fired up the system ... which of course wouldn't boot, so I pulled the drive and then started poking and found a How To Decrypt.html file:

> Your documents, photos, databases and other important files have been encrypted with strongest encryption and unique key, generated for this computer.
> Private decryption key is stored on a secret Internet server and nobody can decrypt your files until you pay and obtain the private key.
> If you see the main locker window, follow the instructions on the locker.
> Overwise, it's seems that you or your antivirus deleted the locker program.
> Now you have the last chance to decrypt your files.
>
> Open http://43qzvceo6ondd6wt.onion.cab or http://43qzvceo6ondd6wt.tor2web.org in your browser. They are public gates to the secret server.
>
> If you have problems with gates, use direct connection:
>
> 1. Download Tor Browser from http://torproject.org.
> 2. In the Tor Browser open the http://43qzvceo6ondd6wt.onion
>    Note that this server is available via Tor Browser only.
>    Retry in 1 hour if site is not reachable.
>
> Copy and paste the following public key in the input form on server. Avoid missprints.
>
>     YK3J5JD-TLOKGZ3-PLQHB3W-WX7J5TN-7BKTWDA-BS3XN6E-WLDVMGE-G6YTIPP
>     2FHQFNA-MVTSWCN-JQYVBNR-MKWSBME-RATZWZB-QRDVSLW-IQGXFDE-GERTAIH
>     GJZCGUG-NWFWMXO-JNG2HRV-NHFFYXN-RRO6N6J-OABS5JE-XLNYIA2-NF5PZXG
>
> Follow the instructions on the server.
>
> The list of your encrypted files:

If you go to the tor site, you're presented with a block that asks you to paste in the key and then gives you the option to decrypt 2 files for free to prove they just want $750.  A little bit of Googling turned up [the following link](http://www.pcrisk.com/removal-guides/8120-your-personal-files-are-encrypted-virus) basically saying all hope is lost.

### Similarities
I remember an email that I got that looked similar and poked around for it. I found that it had 2 attachments, one worthless and one Microsoft Doc file. Heading over to [virustotal with the file](https://www.virustotal.com/en/file/1d3ca0a972e788bd6e693d4a08ea80d228ebe4b4a56a3ff47cfd025bfd607999/analysis/) shows a lot of trojan style flags from my file ... still looking to get a hold of hers. Based on the file and quick walk through binwalk and a quick look at the strings, I think this is the culprite:

    [wyatt@lazarus:~/Downloads/ransomware]$ binwalk -e label_52740071.doc

    DECIMAL   	HEX       	DESCRIPTION
    -------------------------------------------------------------------------------------------------------------------
    8372      	0x20B4    	Zip archive data, at least v2.0 to extract, compressed size: 255,  uncompressed size: 540, name: "[Content_Types].xml"
    8676      	0x21E4    	Zip archive data, at least v2.0 to extract, compressed size: 192,  uncompressed size: 310, name: "_rels/.rels"
    8909      	0x22CD    	Zip archive data, at least v2.0 to extract, compressed size: 131,  uncompressed size: 138, name: "theme/theme/themeManager.xml"
    9098      	0x238A    	Zip archive data, at least v2.0 to extract, compressed size: 1776,  uncompressed size: 6846, name: "theme/theme/theme1.xml"
    10926     	0x2AAE    	Zip archive data, at least v2.0 to extract, compressed size: 182,  uncompressed size: 283, name: "theme/theme/_rels/themeManager.xml.rels"
    11526     	0x2D06    	End of Zip archive
    11548     	0x2D1C    	XML document, version: "1.0"

Files:

    [wyatt@lazarus:~/Downloads/ransomware]$ find .
    .
    ./_label_52740071.doc.extracted
    ./_label_52740071.doc.extracted/themeManager.xml.zip
    ./_label_52740071.doc.extracted/[Content_Types].xml.zip
    ./_label_52740071.doc.extracted/theme1.xml.zip
    ./_label_52740071.doc.extracted/_rels
    ./_label_52740071.doc.extracted/_rels/.rels
    ./_label_52740071.doc.extracted/2D1C.xml
    ./_label_52740071.doc.extracted/themeManager.xml.rels.zip
    ./_label_52740071.doc.extracted/.rels.zip
    ./_label_52740071.doc.extracted/theme
    ./_label_52740071.doc.extracted/theme/theme
    ./_label_52740071.doc.extracted/theme/theme/themeManager.xml
    ./_label_52740071.doc.extracted/theme/theme/_rels
    ./_label_52740071.doc.extracted/theme/theme/_rels/themeManager.xml.rels
    ./_label_52740071.doc.extracted/theme/theme/theme1.xml
    ./_label_52740071.doc.extracted/[Content_Types].xml
    ./label_52740071.doc

Interesting Strings for **2D1C.xml** (there's a lot more, but these popped out to me):

    PROJECT.THISDOCUMENT.H
    PROJECT.THISDOCUMENT.AUTOOPEN
    PROJECT.THISDOCUMENT.FINDTEST
    PROJECT.THISDOCUMENT.AUTO_OPEN
    c:\Users\
    \AppData\Local\Temp

### Digging Deeper
[@JElchison](https://github.com/JElchison):

> confirmed that the landing page is identical per public key, i also ended up at http://43qzvceo6ondd6wt.onion/payment?id=28aa15bb5241fe2054c3a74a112400a4
>
> i'll bet the "id" field is the key in a database table that contains the public/private keys, with possible bindings to bitcoin address.  but...
>
> the public key appears to be presented in base32, but the math doesn't work out:
>
>     (7 characters * 8 columns * 3 rows) * 5 bits/character for base32 = 840-bit key
>
> since this is nonstandard, i'll bet there's additional data included, such as that "id" or (maybe, if we're lucky, the private key itself).
>
> if the virus is self-propagating, then it must do key generation on the fly, which means that it needs to transmit the private key to the server somehow.  the lazy way to do this is to just include it with the "public key" that the web app takes as input.  doing so ensures that they can always accept money, even if the private key never made it to the server beforehand.  also, this makes the onion node stateless (no database), which is attractive for hiding from law enforcement.
>
> on second thought, it's unlikely that the private key could be sent over the wire, without ready access to tor network.  so, likely one of these is true:
>
>  * virus is NOT self-replicating, and sends ONLY the public key to the victim
>  * virus is self-replicating, and the private key is included in this base32-encoded blob that the user inputs.
>
> if [@wyattearp](https://github.com/wyattearp) can access the payload, i'd love to reverse the block that prints this "public key" to see how it's made.

The following files were found in C:/Users/Main/AppData/Local/Temp/ and they appear to the actual files we're interested in:

    4ff7a6a944aace86a2534354f67601a73851f3333b99256d5a3d27d5b4a54780  jwmbmfl.exe
    aec4b4872e0a060c9f67f1fc637f54a2a338bf925ea2df9af9748c5e736795fc  update.exe

Virus total provides us info for [update.exe](https://www.virustotal.com/en/file/aec4b4872e0a060c9f67f1fc637f54a2a338bf925ea2df9af9748c5e736795fc/analysis/) that appears to make sense. One of the first hits is **Crypt3.BVYS**

Opening the binary in IDA reveals about what you'd expect, packed all over the place LZMA code. Running it in IDA, it appears that this section of code is where the pop up message becomes displayed:

    .text:00403E1A ; __fastcall __security_check_cookie(x)
    .text:00403E1A @__security_check_cookie@4 proc near    ; CODE XREF: sub_401000+160p
    .text:00403E1A                                         ; sub_40121D+A9Fp ...
    .text:00403E1A
    .text:00403E1A ; FUNCTION CHUNK AT .text:004041B0 SIZE 00000009 BYTES
    .text:00403E1A
    .text:00403E1A                 cmp     ecx, ds:___security_cookie

After running, it not only encrypts files, but it does the following other items (at least it appears to have the capability to do so):
  * Install fake root CAs: /CN=www.xl4wvq5mrqdmhg.net
  * Has a crap ton of python: <tr><td>C:\Python27\Lib\test\</td><td>https_svn_python_org_root.PEM</td></tr>
  * We also see that Stepan is a dick?:
    * f:\stepan\openssl/ssl/certs
    * f:\stepan\openssl/ssl/cert.pem
  * It has it's own TOR client: Sina Rabbani (inf0)
  * Appears to turn off / disable A/V & firewall (or at least encourages such)
  * Calls out to ip.telize.com to log where you're coming from
  * Takes and sends off the picture of your "user" for your Window's login

These look to be close to the "real" function names:

    tiscreentext
    titextfile
    tihtmlfile
    tiwelcome
    tiinfopage
    tisearchfiles
    tifoundfiles
    titestdone
    tinofoundfiles
    tirequestkey
    tireqfailed
    tioffline
    tipaid
    tipayment
    tisupport
    tisupportpage
    tisupportsend
    tiexchange
    tidecrypting
    tidecryptdone
    titimeexpired
