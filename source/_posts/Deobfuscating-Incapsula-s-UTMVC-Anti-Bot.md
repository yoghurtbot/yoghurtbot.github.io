---
title: Deobfuscating Imperva's utmvc Anti-Bot Script
date: 2023-03-04 23:08:36
tags:
---

# **Introduction**

In this post, we're going take a deep dive into the utmvc challenge script that is part of the FlexProtect suite (formerly called Incapsula) by Imperva. The utmvc challenge is one of two challenge scripts that is presented to a users browser when browsing a protect site, the second being reese84. Depending on the setup of the site, the users browser will need to solve both the utmvc and reese84 challenge, and some sites will require only the reese84 challenge. It seems that the utmvc challenge is somewhat of a "legacy" challenge, and has been superceeded by the reese84 script.

The goal of this article is to fully de-obfuscate the utmvc script so that we can understand what it is doing, which can ultimately lead us to build a generator to mock/spoof the values the script requires.

This article is going to use various well known Javascript de-obfuscation techniques, including [BabelJS](https://babeljs.io/). Unforunately the documentation for BabelJS is somewhat non-existent, so I will do my best to explain what is going on throughout this article. If you are new to BabelJS, I highly reccomend [this excellent series](https://steakenthusiast.github.io/) by PianoMan.

Ready? Lets begin.

## Finding the script

For the purpose of this article, I am going to use the utmvc script that is served up on [SmythsToys](https://www.smythstoys.com/). Firstly lets open up [Charles Proxy](https://www.charlesproxy.com/)  so that we can inspect the network traffic. Then open up a browser, clear cookies, cache, local storage and then browse to our target website, we can see a few requests to Incapsula releated resources. The one that looks of interest to us here is the `POST` to `_Incapsula_Resource`. If we have a look at the HTTP contents, we can see that the `___utmvc` cookie is sent to this endpoint.

![Screenshot 2023-03-04 at 23.50.22](/Users/tom/Desktop/Screenshot 2023-03-04 at 23.50.22.png)

If we have a look at the `GET` request to the same endpoint, we see something interesting...

![Screenshot 2023-03-04 at 23.55.18](/Users/tom/Desktop/Screenshot 2023-03-04 at 23.55.18.png)

This Javascript code is the script that generates the `___utmvc` cookie. 

The code looks like this *(truncated for abrevity)*:

```js
(function() {	
	var z = "";
	var b = "766172205f3.....";
	eval((function() {
		for (var i = 0; i < b.length; i += 2) {
			z += String.fromCharCode(parseInt(b.substring(i, i + 2), 16));
		}
		return z;
	})());
})();
```

Lets break it down.

What we have here is an Immeditely Invoke Function Expression (IIFE). Basically this block of code will be immediately executed when a user visits the webpage. Inside the IIFE, we have a variable called `b` which contains a bunch of character codes. We then have a nested IIFE which is wrapped around an `eval` call. The IIFE inside the `eval` is doing a very basic string decoder function. It's basicallhy looping through the characters in `b`, 2 at a time, and appending their ASCII equivilent values to the `z` variable.

## Decoding the script

Now we have the code, we need to turn it into tangible Javascript code. Lets take the code from above and make a simpler decoder than will print out the decoded string to our console.

```js
var b = "766172205f3...."
var final = "";
for (var i = 0; i < b.length; i += 2) {
  final += String.fromCharCode(parseInt(b.substring(i, i + 2), 16));
}
console.log(final);
```

So in our console we should now have the decoded string. It looks a bit scary at first, but if we run the output through a [JS beautifier](https://beautifier.io/), we get something that resembles somewhat recogniseable Javascript:

```js
(function(_0x246fbc, _0x279367) {
    var _0x3ef8fa = function(_0x4e4813) {
        while (--_0x4e4813) {
            _0x246fbc['\x70\x75\x73\x68'](_0x246fbc['\x73\x68\x69\x66\x74']());
        }
    };
    var _0x2247fc = function() {
        var _0x3108b4 = {
            '\x64\x61\x74\x61': {
                '\x6b\x65\x79': '\x63\x6f\x6f\x6b\x69\x65',
                '\x76\x61\x6c\x75\x65': '\x74\x69\x6d\x65\x6f\x75\x74'
            },
            '\x73\x65\x74\x43\x6f\x6f\x6b\x69\x65': function(_0x564ed7, _0x16e7cb, _0x1a47f2, _0x3f5a26) {
                _0x3f5a26 = _0x3f5a26 || {};
                var _0x7c508a = _0x16e7cb + '\x3d' + _0x1a47f2;
                var _0x367e08 = 0x0;
                for (var _0x367e08 = 0x0, _0x492c74 = _0x564ed7['\x6c\x65\x6e\x67\x74\x68']; _0x367e08 < _0x492c74; _0x367e08++) {
                    var _0x2c0fa7 = _0x564ed7[_0x367e08];
                    _0x7c508a += '\x3b\x20' + _0x2c0fa7;
                    var _0x4478c3 = _0x564ed7[_0x2c0fa7];
                    _0x564ed7['\x70\x75\x73\x68'](_0x4478c3);
                    _0x492c74 = _0x564ed7['\x6c\x65\x6e\x67\x74\x68'];
                    if (_0x4478c3 !== !![]) {
                        _0x7c508a += '\x3d' + _0x4478c3;
                    }
                }
                _0x3f5a26['\x63\x6f\x6f\x6b\x69\x65'] = _0x7c508a;
            },
            '\x72\x65\x6d\x6f\x76\x65\x43\x6f\x6f\x6b\x69\x65': function() {
                return '\x64\x65\x76';
            },
....
```

Looking at the above, we can see that we have a few obfuscation techniques going on:

- Strings are protected by hexadecimal escape sequences (`\x` notation)
- Variable names have been renamed to hexadecimal
- Numbers have been replaced by their hexadecimal equivielent

Just from experience, I know that these particular techniques are a result of [ObfuscatorIO](https://obfuscator.io/).

## Revealing Strings

As I mentioned previously, the strings are being represented by hexadecimal escape sequences, so we need to reveal and replaice them with their plaintext values. Luckily Babel provides a really easy way for us to do this. The plaintext value can be found in the `extra` property of the `StringLiteral` type:

<img src="/Users/tom/Documents/Dev Projects/blog/blog/images/Screenshot 2023-03-05 at 00.21.12.png" alt="Screenshot 2023-03-05 at 00.21.12" style="zoom:50%;" />

And we can write a simple Babel visitor for this:

```js
const t = require("@babel/types");
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

const ast = parser.parse(decoded);

const revealStrings = {
  StringLiteral(path){
    if (!path.node.extra) return;
    path.replaceWith(t.stringLiteral(path.node.extra.rawValue))
  }
}
traverse(ast, revealStrings);

```

After we have run the above visitor, we've now reveleaed the strings! (kind of, I'll explain later)
```js
 var _0x2247fc = function () {
    var _0x3108b4 = {
      "data": {
        "key": "cookie",
        "value": "timeout"
      },
      "setCookie": function (_0x564ed7, _0x16e7cb, _0x1a47f2, _0x3f5a26) {
        _0x3f5a26 = _0x3f5a26 || {};
        var _0x7c508a = _0x16e7cb + "=" + _0x1a47f2;
        var _0x367e08 = 0x0;
        for (var _0x367e08 = 0x0, _0x492c74 = _0x564ed7["length"]; _0x367e08 < _0x492c74; _0x367e08++) {
          var _0x2c0fa7 = _0x564ed7[_0x367e08];
          _0x7c508a += "; " + _0x2c0fa7;
          var _0x4478c3 = _0x564ed7[_0x2c0fa7];
          _0x564ed7["push"](_0x4478c3);
          _0x492c74 = _0x564ed7["length"];
          if (_0x4478c3 !== !![]) {
            _0x7c508a += "=" + _0x4478c3;
```

So now we have all the strings in a readable format, but if we have a look in the script, we can still see that there are Base64 strings all over the place. Whilst some of the strings are now readable, there are still a lot of strings that aren't. Throughout the script there are repeated occurances of this type of stuff: `_0xe3ea("0x44", "@7QD")`, `_0xe3ea("0x4f", "E@5e")` - a call expression with two parameters, a number rerpesented as a hex string, and a string literal. 

If we take a look at the function `_0xe3ea` we can see references to "rc4". RC4 is a symmetric stream cipher algorithm that is used for encrypting and decrypting strings. Again from experience, I know that RC4 string encryption is a feature of ObfuscatorIO. So based on this knowledge, can we assume that the `_0xe3ea` is an RC4 decoder function? Yes! But let me explain how it works.

At the top of the script we have a `VariableDeclaration` which is initiialized to an `ArrayExpression`. This again is a feature of ObfuscatorIO and is used to make reading strings harder. When you run code through ObfuscatorIO, it will take the strings, encrypt them using RC4, put them into an array and used a 'decoder' function which will fetch the string from the array. In this case, we have an array of encrypted strings that get run through the RC4 decryption function. We can write a visitor to store the values in an array:

```js
const t = require("@babel/types");
const traverse = require("@babel/traverse").default;

let wordArray = []
const getWordArray = {
  Program(path){ 
    wordArray = node.body[0].declarations[0].init.elements.map((val => {
      return val.value;
    }));
    path.stop();
  }
}
traverse(ast, getWordArry);
```

We now have an array of RC4 encrypted strings.



## Understanding the RC4 String Encryption

### Word Array and Shuffle

Before we get too far into the the RC4  encryption, we need to understand how the RC4 encrypted strings are stored in the script. Just above the RC4 decryption function, we have another IIFE:

```js
(function (_0x246fbc, _0x279367) {
  var _0x3ef8fa = function (_0x4e4813) {
    while (--_0x4e4813) {
      _0x246fbc["push"](_0x246fbc["shift"]());
    }
  };
  var _0x2247fc = function () {
    var _0x3108b4 = {
      "data": {
        "key": "cookie",
        "value": "timeout"
      },
      "setCookie": function (_0x564ed7, _0x16e7cb, _0x1a47f2, _0x3f5a26) {
        _0x3f5a26 = _0x3f5a26 || {};
        var _0x7c508a = _0x16e7cb + "=" + _0x1a47f2;
        var _0x367e08 = 0x0;
        for (var _0x367e08 = 0x0, _0x492c74 = _0x564ed7["length"]; _0x367e08 < _0x492c74; _0x367e08++) {
          var _0x2c0fa7 = _0x564ed7[_0x367e08];
          _0x7c508a += "; " + _0x2c0fa7;
          var _0x4478c3 = _0x564ed7[_0x2c0fa7];
          _0x564ed7["push"](_0x4478c3);
          _0x492c74 = _0x564ed7["length"];
          if (_0x4478c3 !== !![]) {
            _0x7c508a += "=" + _0x4478c3;
          }
        }
        _0x3f5a26["cookie"] = _0x7c508a;
      },
      "removeCookie": function () {
        return "dev";
      },
      "getCookie": function (_0x230827, _0x3e3448) {
        _0x230827 = _0x230827 || function (_0x3772b8) {
          return _0x3772b8;
        };
        var _0x26af38 = _0x230827(new RegExp("(?:^|; )" + _0x3e3448["replace"](/([.$?*|{}()[]\/+^])/g, "$1") + "=([^;]*)"));
        var _0x4fd28d = function (_0x53e56d, _0xd00715) {
          _0x53e56d(++_0xd00715);
        };
        _0x4fd28d(_0x3ef8fa, _0x279367);
        return _0x26af38 ? decodeURIComponent(_0x26af38[0x1]) : undefined;
      }
    };
    var _0x255171 = function () {
      var _0x5b8adf = new RegExp("\\w+ *\\(\\) *{\\w+ *['|\"].+['|\"];? *}");
      return _0x5b8adf["test"](_0x3108b4["removeCookie"]["toString"]());
    };
    _0x3108b4["updateCookie"] = _0x255171;
    var _0xecc8a7 = "";
    var _0x586381 = _0x3108b4["updateCookie"]();
    if (!_0x586381) {
      _0x3108b4["setCookie"](["*"], "counter", 0x1);
    } else if (_0x586381) {
      _0xecc8a7 = _0x3108b4["getCookie"](null, "counter");
    } else {
      _0x3108b4["removeCookie"]();
    }
  };
  _0x2247fc();
})(_0x3eae, 0x14a);
```

This time the IFFE has two parameters: `_0x3eae` and `0x14a`. `_0x3eae` is the word array that we extracted above (an `ArrayExpression` of encrypted strings) and `0x14a` is a literal that has a numeric value of `330`. This number is important. The array of strings is out of order and this number is the 'magic number' which is used to shuffle the array to get them back in the correct order.

This is the shuffle code from the script:

```js
var _0x3ef8fa = function (_0x4e4813) {
  while (--_0x4e4813) {
    _0x246fbc["push"](_0x246fbc["shift"]());
  }
};
```

But it can be re-written to something a bit easier to read:

```js
function shuffleWordArray(array, num) {
  function performShift(times) {
    while (--times) {
      array.push(array.shift());
    }
  }
  performShift(++num);
}
```

We now need to shuffle our encrypted string array by the magic number to ensure that the strings are in the correct order in the array:

```js
//shuffle the array to ensure it's in the correct order
shuffleWordArray(wordArray, magicNumber);

//wordArray is now in the correct order
```

### RC4 Deep Dive

This is the entire RC4 decoder function, I've added some comments to make it easier to understand:

```js
var _0xe3ea = function (_0x246fbc, _0x279367) {
  //Useless junk
  _0x246fbc = _0x246fbc - 0x0;
  //_0x3eae - This is our huge word array at the top of the script which contains all the RC4 encrypted strings
  //_0x246fbc - This is an index getter for the array
  var _0x3ef8fa = _0x3eae[_0x246fbc];
  
  if (_0xe3ea["initialized"] === undefined) {
    (function () {
      var _0x45df7b = Function("return (function () " + "{}.constructor(\"return this\")()" + ");");
      var _0x2247fc = _0x45df7b();
      var _0x3108b4 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=";
      _0x2247fc["atob"] || (_0x2247fc["atob"] = function (_0x564ed7) {
        var _0x16e7cb = String(_0x564ed7)["replace"](/=+$/, "");
        for (var _0x1a47f2 = 0x0, _0x3f5a26, _0x7c508a, _0x76115b = 0x0, _0x367e08 = ""; _0x7c508a = _0x16e7cb["charAt"](_0x76115b++); ~_0x7c508a && (_0x3f5a26 = _0x1a47f2 % 0x4 ? _0x3f5a26 * 0x40 + _0x7c508a : _0x7c508a, _0x1a47f2++ % 0x4) ? _0x367e08 += String["fromCharCode"](0xff & _0x3f5a26 >> (-0x2 * _0x1a47f2 & 0x6)) : 0x0) {
          _0x7c508a = _0x3108b4["indexOf"](_0x7c508a);
        }
        return _0x367e08;
      });
    })();
    //The actual RC4 encryption code, parameter 1 is the encrypted string, parameter 2 is the key
    var _0x492c74 = function (_0x2c0fa7, _0x4478c3) {
      var _0x230827 = [],
        _0x3e3448 = 0x0,
        _0x3772b8,
        _0x26af38 = "",
        _0x4fd28d = "";
      _0x2c0fa7 = atob(_0x2c0fa7);
      for (var _0x53e56d = 0x0, _0xd00715 = _0x2c0fa7["length"]; _0x53e56d < _0xd00715; _0x53e56d++) {
        _0x4fd28d += "%" + ("00" + _0x2c0fa7["charCodeAt"](_0x53e56d)["toString"](0x10))["slice"](-0x2);
      }
      _0x2c0fa7 = decodeURIComponent(_0x4fd28d);
      for (var _0x255171 = 0x0; _0x255171 < 0x100; _0x255171++) {
        _0x230827[_0x255171] = _0x255171;
      }
      for (_0x255171 = 0x0; _0x255171 < 0x100; _0x255171++) {
        _0x3e3448 = (_0x3e3448 + _0x230827[_0x255171] + _0x4478c3["charCodeAt"](_0x255171 % _0x4478c3["length"])) % 0x100;
        _0x3772b8 = _0x230827[_0x255171];
        _0x230827[_0x255171] = _0x230827[_0x3e3448];
        _0x230827[_0x3e3448] = _0x3772b8;
      }
      _0x255171 = 0x0;
      _0x3e3448 = 0x0;
      for (var _0x5b8adf = 0x0; _0x5b8adf < _0x2c0fa7["length"]; _0x5b8adf++) {
        _0x255171 = (_0x255171 + 0x1) % 0x100;
        _0x3e3448 = (_0x3e3448 + _0x230827[_0x255171]) % 0x100;
        _0x3772b8 = _0x230827[_0x255171];
        _0x230827[_0x255171] = _0x230827[_0x3e3448];
        _0x230827[_0x3e3448] = _0x3772b8;
        _0x26af38 += String["fromCharCode"](_0x2c0fa7["charCodeAt"](_0x5b8adf) ^ _0x230827[(_0x230827[_0x255171] + _0x230827[_0x3e3448]) % 0x100]);
      }
      return _0x26af38;
    };
    _0xe3ea["rc4"] = _0x492c74;
    _0xe3ea["data"] = {};
    _0xe3ea["initialized"] = !![];
  }
  var _0xecc8a7 = _0xe3ea["data"][_0x246fbc];
  if (_0xecc8a7 === undefined) {
    if (_0xe3ea["once"] === undefined) {
      var _0x586381 = function (_0x3328ca) {
        this["rc4Bytes"] = _0x3328ca;
        this["states"] = [0x1, 0x0, 0x0];
        this["newState"] = function () {
          return "newState";
        };
        this["firstState"] = "\\w+ *\\(\\) *{\\w+ *";
        this["secondState"] = "['|\"].+['|\"];? *}";
      };
      _0x586381["prototype"]["checkState"] = function () {
        var _0x17ea7d = new RegExp(this["firstState"] + this["secondState"]);
        return this["runState"](_0x17ea7d["test"](this["newState"]["toString"]()) ? --this["states"][0x1] : --this["states"][0x0]);
      };
      _0x586381["prototype"]["runState"] = function (_0x4355f3) {
        if (!Boolean(~_0x4355f3)) {
          return _0x4355f3;
        }
        return this["getState"](this["rc4Bytes"]);
      };
      _0x586381["prototype"]["getState"] = function (_0x27ff93) {
        for (var _0x55a624 = 0x0, _0xa7b380 = this["states"]["length"]; _0x55a624 < _0xa7b380; _0x55a624++) {
          this["states"]["push"](Math["round"](Math["random"]()));
          _0xa7b380 = this["states"]["length"];
        }
        return _0x27ff93(this["states"][0x0]);
      };
      new _0x586381(_0xe3ea)["checkState"]();
      _0xe3ea["once"] = !![];
    }
    _0x3ef8fa = _0xe3ea["rc4"](_0x3ef8fa, _0x279367);
    _0xe3ea["data"][_0x246fbc] = _0x3ef8fa;
  } else {
    _0x3ef8fa = _0xecc8a7;
  }
  return _0x3ef8fa;
};
```

Inside the RC4 code, we can see this: `_0xe3ea["rc4"] = _0x492c74;`. What this tells us is that the the actual RC4 decryption function is defined in `_0x492c74`. If we have a look at `_0x492c74` we can see that it takes two parameters; lets assume the first parameter is the encrypted string and the second parameter is the decryption key.  Next we can isolate the RC4 function and pull it out so we can use it as a a standalone piece of code later. Again we can write a simple Babel plugin to extract the function:

```js
const t = require("@babel/types");
const traverse = require("@babel/traverse").default;

let rc4Function;
const findRc4Function = {
  AssignmentExpression(path){ 
    const { node } = path;
    
    // Find the '_0xe3ea["rc4"] = _0x492c74;' node
    if (t.isMemberExpression(node.left) &&
       node.left.property.value === "rc4"
     ) {
      // Go back 1 node and get the RC4 function
      rc4Function = path.getStatementParent().getPrevSibling();
      path.stop();
    }
  }
}

traverse(ast, findRc4Function);
```

We now have the RC4 decryption function isolated by itself. If you re-call from earlier, I said that we saw this type of code all over the place: `_0xe3ea("0x44", "@7QD")`. We've now discovered that the RC4 decrpytion function is nested inside `_0xe3ea`, and `_0xe3ea` is a "getter" that internally calls the RC4 function and spits out the decrypted value. At the top of the getter we can see a reference with an index to our word array (`_0x3eae`). What this code is doing is fetching the encrypted string from our array, and then sending it to the RC4 decrypter. This is why it was important for us to shuffle the array before we got to this stage, otherwise we'd get the wrong decrypted string back!

```js
var _0xe3ea = function (_0x246fbc, _0x279367) {
  _0x246fbc = _0x246fbc - 0x0;
  var _0x3ef8fa = _0x3eae[_0x246fbc]; // << This is a reference to the word arary with an index
```

### Putting it together

We now have all the information we need to create our next Babel visitor, which will be used to replace the encrypted strings. We have the following knowledge:

- An array of encrypted strings (shuffled to be in the correct order)
- A reference to the RC4 function
- The identifier name of the `VariableDeclarator` which contains the array of strings
- The "getter" fuction identifier which calls the RC4 function internally

We can use `vm` to create a small context spearate from our main code, which will execute and return a valiue. We're going to setup a context which contains just the RC4 code, call it from our visitor, and do a replacement:

```js
const t = require("@babel/types");
const generate = require("@babel/generator").default;
const vm = require("vm");

function atob(a) {
	return new Buffer(a, 'base64').toString('binary');
};

const revealStringsVisitor = {
	AssignmentExpression(path){
      const { node } = path;

      if (
          t.isMemberExpression(node.left) &&
          node.left.property.value === "rc4"
      ){
          // This is a variable to a function, i.e.  var _0x1cb434 = function (_0xc7576, _0x14cfd3) { .. rc4 stuff
          const rc4FuncVariableDeclaration = path.getStatementParent().getPrevSibling();
          // This is a "getter" that calls the RC4 function, i.e. the outer function
          const getterFunction = rc4FuncVariableDeclaration.getFunctionParent().getStatementParent()

          // The word array identifier
          // var _0x3ef8fa = _0x3eae[_0x246fbc];
          const arrayName = getterFunction.get("declarations.0.init.body").node.body[1].declarations[0].init.object.name;
          const decoderFuncName = getterFunction.node.declarations[0].id.name;

          //Setup a context with atob
          const ctx = {atob: atob}
          vm.runInNewContext(generate(rc4FuncVariableDeclaration.node).code, ctx);

          const binding = path.scope.getBinding(decoderFuncName);

          // Find all references to the decoder, i.e. _0xe3ea("0x44", "@7QD")
          for(const ref of binding.referencePaths){
              if (
                  t.isIdentifier(ref.parentPath.node.callee) &&
                  t.isCallExpression(ref.parentPath) &&
                  ref.parentPath.node.arguments.length === 2 &&
                  ref.parentPath.node.arguments[0].type === "StringLiteral" &&
                  ref.parentPath.node.arguments[1].type === "StringLiteral"
              ) {
                  // Index is the first parameter, and lets conver to an int
                  const index = parseInt(ref.parentPath.node.arguments[0].value);

                  // Key is the 2nd parameter
                  const key = ref.parentPath.node.arguments[1].value;

                  // Call the RC4 decryption function inside the context, and get the output
                  const decryptedValue = ctx[rc4FuncVariableDeclaration.node.declarations[0].id.name](this.wordArrays[arrayName][index], key)

                  // Do a plain text replacement
                  ref.parentPath.replaceWith(t.valueToNode(decryptedValue));
              }
          }
      }
  } 
}
```

After running this plugin, we now have all strings revelaed!

```js
...  
var _0x19a5d8 = this["window"];
var _0x145c56 = _0x19a5d8["document"];
var _0xc68334 = "";
var _0x641c3c = "";
if (_0x2f96a5["yEQ"](typeof _0x19a5d8["console"], "undefined")) {
  _0xc68334 = _0x19a5d8["console"];
  _0x641c3c = _0xc68334["log"];
}
var _0x2b232b = _0x19a5d8["navigator"];
var _0x35da0a = _0x19a5d8["encodeURIComponent"];
var _0x3ed2cd = new _0x19a5d8["Date"]()["getTime"]();
...
```



