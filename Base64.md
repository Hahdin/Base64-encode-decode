```js
/**
 * The "Unicode Problem"
Since DOMStrings are 16-bit-encoded strings, in most browsers calling window.btoa on a
Unicode string will cause a Character Out Of Range exception if a character exceeds the range of a 8-bit byte (0x00~0xFF).

JavaScript's UTF-16 => base64
-----------------------------
A very fast and widely useable way to solve the unicode problem is by encoding JavaScript native UTF-16 strings directly into base64.
This method is particularly efficient because it does not require any type of conversion, except mapping a string into an array. 

The following code is also useful to get an ArrayBuffer from a Base64 string and/or viceversa

JavaScript's UTF-16 => UTF-8 => base64
--------------------------------------
Setting UTF8 to true converts JavaScript's native UTF-16 string into a UTF-8 string and then encoding the latter into base64.
This also grants that converting a pure ASCII string to base64 always produces the same output as the native btoa().

JavaScript's UTF-16 => binary string => base64
----------------------------------------------
atobUTF16/btoaUTF16 is the fastest and most compact possible approach. 
Then, instead of rewriting atob() and btoa() it uses the native ones.
This is made possible by the fact that instead of using typed arrays as encoding/decoding inputs
this uses binary strings as an intermediate format. It is a “dirty” workaround in comparison to the default use
as binary strings are a grey area, however it works pretty well and requires only a few extra lines of code.
 */

"use strict";

class EncodeDecode {
  constructor(encodeThis, UTF8 = false) {
    this.stringToEncode = encodeThis;
    this.UTF8 = UTF8;
  }
  /**
   * Getters
   */
  get theString() {
    return this.stringToEncode;
  }
  get theEncodedString() {
    return this.encodedString;
  }
  get theDecodedString() {
    return this.decodedString;
  }
  /** 
   *  Base64 string to array encoding
   */

  /**
   * Base64 string to array encoding
   * @param {UINT} nUint6 
   */
  uint6ToB64(nUint6) {
    return nUint6 < 26 ?
      nUint6 + 65
      : nUint6 < 52 ?
        nUint6 + 71
        : nUint6 < 62 ?
          nUint6 - 4
          : nUint6 === 62 ?
            43
            : nUint6 === 63 ?
              47
              :
              65;
  }
  /**
   * Array of bytes to base64 string decoding
   * @param {number} nChr character code
   */
  b64ToUint6(nChr) {
    return nChr > 64 && nChr < 91 ?
      nChr - 65
      : nChr > 96 && nChr < 123 ?
        nChr - 71
        : nChr > 47 && nChr < 58 ?
          nChr + 4
          : nChr === 43 ?
            62
            : nChr === 47 ?
              63
              :
              0;
  }

  /**
   * Encode an Array
   * @param {array} aBytes Uint8Array buffer
   */
  base64EncArr(aBytes) {
    let eqLen = (3 - (aBytes.length % 3)) % 3, sB64Enc = "";
    for (let nMod3, nLen = aBytes.length, nUint24 = 0, nIdx = 0; nIdx < nLen; nIdx++) {
      nMod3 = nIdx % 3;
      const uint6ToB64 = this.uint6ToB64;
      /* Uncomment the following line in order to split the output in lines 76-character long: */
      /*
      if (nIdx > 0 && (nIdx * 4 / 3) % 76 === 0) { sB64Enc += "\r\n"; }
      */
      nUint24 |= aBytes[nIdx] << (16 >>> nMod3 & 24);
      if (nMod3 === 2 || aBytes.length - nIdx === 1) {
        sB64Enc += String.fromCharCode(uint6ToB64(nUint24 >>> 18 & 63), uint6ToB64(nUint24 >>> 12 & 63), uint6ToB64(nUint24 >>> 6 & 63), uint6ToB64(nUint24 & 63));
        nUint24 = 0;
      }
      this.encodedString = eqLen === 0 ?
        sB64Enc
        :
        sB64Enc.substring(0, sB64Enc.length - eqLen) + (eqLen === 1 ? "=" : "==");
    }

  }
  /**
   *  Encode to base64 using native UTF-16
   */
  toBase64() {
    if (this.UTF8) {
      this.toBase64UTF8();
      return;
    }
    let aUTF16CodeUnits = new Uint16Array(this.stringToEncode.length);
    const self = this;
    Array.prototype.forEach.call(aUTF16CodeUnits, function (el, idx, arr) { arr[idx] = self.stringToEncode.charCodeAt(idx); });
    this.base64EncArr(new Uint8Array(aUTF16CodeUnits.buffer));
  }

  toBase64UTF8() {
    this.base64EncArr(this.strToUTF8Arr(this.stringToEncode));
  }

  /**
   * Decode the encodedString
   * 
   * @param {number} nBlockSize 
   */
  base64DecToArr(nBlockSize) {
    let
      sB64Enc = this.encodedString.replace(/[^A-Za-z0-9\+\/]/g, ""), nInLen = sB64Enc.length,
      nOutLen = nBlockSize ? Math.ceil((nInLen * 3 + 1 >>> 2) / nBlockSize) * nBlockSize : nInLen * 3 + 1 >>> 2, aBytes = new Uint8Array(nOutLen);

    for (let nMod3, nMod4, nUint24 = 0, nOutIdx = 0, nInIdx = 0; nInIdx < nInLen; nInIdx++) {
      nMod4 = nInIdx & 3;
      nUint24 |= this.b64ToUint6(sB64Enc.charCodeAt(nInIdx)) << 18 - 6 * nMod4;
      if (nMod4 === 3 || nInLen - nInIdx === 1) {
        for (nMod3 = 0; nMod3 < 3 && nOutIdx < nOutLen; nMod3++ , nOutIdx++) {
          aBytes[nOutIdx] = nUint24 >>> (16 >>> nMod3 & 24) & 255;
        }
        nUint24 = 0;
      }
    }

    if (this.UTF8) {
      return this.UTF8ArrToStr(aBytes);
    } else {
      return aBytes.buffer;
    }
  }

  UTF8ArrToStr(aBytes) {
    let sView = "";
    for (let nPart, nLen = aBytes.length, nIdx = 0; nIdx < nLen; nIdx++) {
      nPart = aBytes[nIdx];
      sView += String.fromCharCode(
        nPart > 251 && nPart < 254 && nIdx + 5 < nLen ? /* six bytes */
          /* (nPart - 252 << 30) may be not so safe in ECMAScript! So...: */
          (nPart - 252) * 1073741824 + (aBytes[++nIdx] - 128 << 24) + (aBytes[++nIdx] - 128 << 18) + (aBytes[++nIdx] - 128 << 12) + (aBytes[++nIdx] - 128 << 6) + aBytes[++nIdx] - 128
          : nPart > 247 && nPart < 252 && nIdx + 4 < nLen ? /* five bytes */
            (nPart - 248 << 24) + (aBytes[++nIdx] - 128 << 18) + (aBytes[++nIdx] - 128 << 12) + (aBytes[++nIdx] - 128 << 6) + aBytes[++nIdx] - 128
            : nPart > 239 && nPart < 248 && nIdx + 3 < nLen ? /* four bytes */
              (nPart - 240 << 18) + (aBytes[++nIdx] - 128 << 12) + (aBytes[++nIdx] - 128 << 6) + aBytes[++nIdx] - 128
              : nPart > 223 && nPart < 240 && nIdx + 2 < nLen ? /* three bytes */
                (nPart - 224 << 12) + (aBytes[++nIdx] - 128 << 6) + aBytes[++nIdx] - 128
                : nPart > 191 && nPart < 224 && nIdx + 1 < nLen ? /* two bytes */
                  (nPart - 192 << 6) + aBytes[++nIdx] - 128
                  : /* nPart < 127 ? */ /* one byte */
                  nPart
      );
    }
    return sView;
  }

  strToUTF8Arr(sDOMStr) {

    let aBytes, nChr, nStrLen = sDOMStr.length, nArrLen = 0;

    /* mapping... */

    for (let nMapIdx = 0; nMapIdx < nStrLen; nMapIdx++) {
      nChr = sDOMStr.charCodeAt(nMapIdx);
      nArrLen += nChr < 0x80 ? 1 : nChr < 0x800 ? 2 : nChr < 0x10000 ? 3 : nChr < 0x200000 ? 4 : nChr < 0x4000000 ? 5 : 6;
    }

    aBytes = new Uint8Array(nArrLen);

    /* transcription... */

    for (let nIdx = 0, nChrIdx = 0; nIdx < nArrLen; nChrIdx++) {
      nChr = sDOMStr.charCodeAt(nChrIdx);
      if (nChr < 128) {
        /* one byte */
        aBytes[nIdx++] = nChr;
      } else if (nChr < 0x800) {
        /* two bytes */
        aBytes[nIdx++] = 192 + (nChr >>> 6);
        aBytes[nIdx++] = 128 + (nChr & 63);
      } else if (nChr < 0x10000) {
        /* three bytes */
        aBytes[nIdx++] = 224 + (nChr >>> 12);
        aBytes[nIdx++] = 128 + (nChr >>> 6 & 63);
        aBytes[nIdx++] = 128 + (nChr & 63);
      } else if (nChr < 0x200000) {
        /* four bytes */
        aBytes[nIdx++] = 240 + (nChr >>> 18);
        aBytes[nIdx++] = 128 + (nChr >>> 12 & 63);
        aBytes[nIdx++] = 128 + (nChr >>> 6 & 63);
        aBytes[nIdx++] = 128 + (nChr & 63);
      } else if (nChr < 0x4000000) {
        /* five bytes */
        aBytes[nIdx++] = 248 + (nChr >>> 24);
        aBytes[nIdx++] = 128 + (nChr >>> 18 & 63);
        aBytes[nIdx++] = 128 + (nChr >>> 12 & 63);
        aBytes[nIdx++] = 128 + (nChr >>> 6 & 63);
        aBytes[nIdx++] = 128 + (nChr & 63);
      } else /* if (nChr <= 0x7fffffff) */ {
        /* six bytes */
        aBytes[nIdx++] = 252 + (nChr >>> 30);
        aBytes[nIdx++] = 128 + (nChr >>> 24 & 63);
        aBytes[nIdx++] = 128 + (nChr >>> 18 & 63);
        aBytes[nIdx++] = 128 + (nChr >>> 12 & 63);
        aBytes[nIdx++] = 128 + (nChr >>> 6 & 63);
        aBytes[nIdx++] = 128 + (nChr & 63);
      }
    }
    return aBytes;
  }

  btoaUTF16() {
    if (typeof window === 'undefined') {
      this.encodedString = 'window does not exist in this context, use encodeString method';
      return;
    }
    let aUTF16CodeUnits = new Uint16Array(this.stringToEncode.length);
    const self = this;
    Array.prototype.forEach.call(aUTF16CodeUnits, function (el, idx, arr) { arr[idx] = self.stringToEncode.charCodeAt(idx); });
    this.encodedString = window.btoa(String.fromCharCode.apply(null, new Uint8Array(aUTF16CodeUnits.buffer)));
  }

  atobUTF16() {
    if (typeof window === 'undefined') {
      this.decodedString = 'window does not exist in this context, use decodeString method';
      return;
    }
    let sBinaryString = window.atob(this.encodedString), aBinaryView = new Uint8Array(sBinaryString.length);
    Array.prototype.forEach.call(aBinaryView, function (el, idx, arr) { arr[idx] = sBinaryString.charCodeAt(idx); });
    this.decodedString = String.fromCharCode.apply(null, new Uint16Array(aBinaryView.buffer));
  }

  /**
   * Encode a string as Base64
   */
  encodeString() {
    if (this.UTF8) {
      this.toBase64UTF8();
    } else {
      this.toBase64();
    }
  }

  /**
   * Decode the Base64 encoded string
   */
  decodeString() {
    this.decodedString = this.UTF8 ?
      this.base64DecToArr(2) :
      String.fromCharCode.apply(null, new Uint16Array(this.base64DecToArr(2)));
  }

}


/** 
 * Gensuite example
 */
const ukey = 'C214DB31-5056-8D05-F42546BC91522C78';
const secret = 'CBB5A63A-B667-4703-A4FF92E5A2884A30';
const ourString = `${ukey}${secret}`;
const gsuiteEncode = new EncodeDecode(ourString, true);
gsuiteEncode.encodeString();
console.log(`Encoded String: ${gsuiteEncode.theEncodedString}`);
gsuiteEncode.decodeString();
console.log(`Decoded String: ${gsuiteEncode.theDecodedString}`);
console.log(` equal? : ${gsuiteEncode.theString === gsuiteEncode.theDecodedString}`)


// testing 
const myEncodeDecoder = new EncodeDecode("☸☹☺☻☼☾☿");

myEncodeDecoder.encodeString();
console.log(`Encoded String: ${myEncodeDecoder.theEncodedString}`);
/* Encoded String: OCY5JjomOyY8Jj4mPyY= */
myEncodeDecoder.decodeString();
console.log(`Decoded String: ${myEncodeDecoder.theDecodedString}`);
/* Decoded String: ☸☹☺☻☼☾☿ */
console.log(` equal? : ${myEncodeDecoder.theString === myEncodeDecoder.theDecodedString}`)
/* equal? : true */

// fastest way (front end only, use from browser)
myEncodeDecoder.btoaUTF16();
console.log(`Fast Encoded String: ${myEncodeDecoder.theEncodedString}`);
myEncodeDecoder.atobUTF16();
console.log(`Fast Decoded String: ${myEncodeDecoder.theDecodedString}`);
```
