---
title: So, it turns out that .NET’s Regex are more powerful t̖̱̍ͭ͊h̟ͨͨa̞̖̙̔̇n͇̝͚̤̒́ͨ̐ ̯͖̏̌̔Ị̟̮̱̥̇̐̎͂ͬ͗̒ ̪̹̱͙̘ͦ̉ͪͪͣ̉͊o͕̥̝͇͙ͪ͊ͤ̑̂̽́r͔̭̪̮̟͗̍ͨ͗͛ͣḭ̝̜͈ͫ́g̥̹̥̜̦̓̇̓i̪͕̭̞͛ͯ̓͛̔̾ͫn̘̗a̰̜ͨͪ͊l̩͑̐̐́ͥ̚l̜ͨ͋̈ẙͦ́ ̟̬̬̫͙̤ͭ̚t̳͎̱̗̲́h͔͙̰̬̊̈́͊̾o͉ͫ̌̄u͉̲̥g̏ͥ̑̅̽̇h̻͇̥̰̯ͥͯṱ̯̏̄̒͒ͫ̃.͖̟͍̘̼̼̍̐̀͊̓́…
date: 2011-05-10
---

Today, thanks to user [Lucero](http://stackoverflow.com/users/88558/lucero) on StackOverflow, I learned about .NET’s “Balancing Groups” Regex feature.

Basically, any time you use a named capturing group, it actually pushes the capture onto a named stack. You can then pop this stack by using the same capturing group prefixed with a hyphen, like `(?<-stackToPop>)`.

---

Of course, anyone who finds themselves in this situation is going to ask: _can it match XML?_

It’s possible that I am missing something completely (it is rather late at night), but … _very nearly_. I haven’t quite figured out a nested section in the local DTD subset, but no one uses that feature anyway. (Can you spot it?)

Aside from that, most of the well-formedness criteria are handled (the obvious one being element nesting). Things that require non-local information such as entities aren’t handled. I think it is possible to handle duplicate attribute names in this form as well (via a lookahead for duplicate names).

Here is the code, and a test file which shows some stuff that is caught by this. Breaking any of the elements should make it fail:

```csharp
var surrogate = @"([\ud800-\udbff][\udc00-\udfff])";// .NET can't handle \U10000-\u10FFFF
var c = @"([\u0009\u000a\u000d\u0020-\ud7ff\ue000-\ufffd]|"+surrogate + ")"; 
var s = @"([\u0020\u0009\u000d\u000a]+)";
var nameStartChar = @"([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|" + surrogate + ")";
var nameChar = "(" + nameStartChar + @"|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])";
var name = "(?'name'" + nameStartChar + nameChar + "*)";
var names = "(?'names'" + name + @"(\u0020" + name +")*)";
var nmtoken = "(?'nmtoken'" + nameChar + "+)";
var nmtokens = "(?'nmtokens'" + nmtoken + @"(\u0020" + nmtoken +")*)";
var pereference = "%" + name + ";";
var entityReference= "(?'entityRef'&" + name + ";)";
var charref = @"&\#([0-9]+|x[0-9a-fA-F]+);";
var reference = "(?'reference'"+ entityReference + "|" + charref + ")";
var entityValue = "(?'entityValue'\"([^%&\"]|" + pereference + "|" + reference + ")*\"|'([^%&']|" + pereference + "|" + reference + ")*')";
var eq = "(?'eq'" + s + "?=" + s + "?)";
var versionNum  = @"1\.[0-9]+";
var comment = "(?'comment'<!--((?!--)" + c + ")*-->)";
var PITarget = "(?'pitarget'(?![xX][mM][lL])"+name+")";
var PI= @"(?'PI'<\?" + PITarget + "(" + s + @"((?!\?>)" + c + @")*)?\?>)";
var misc = "(?'misc'" + comment + "|" + PI + "|" + s + ")";
var versionInfo = "(?'versionInfo'"+ s + "version" + eq + "('" + versionNum + "'|\"" + versionNum + "\"))";
var encName = "(?'encName'[A-Za-z][A-Za-z0-9._-]*)";
var encodingDecl = "(?'encodingDecl'" + s + "encoding" + eq + "(\"" + encName + "\"|'"+ encName + "'))";
var sddecl = "(?'sddecl'" + s + "standalone" + eq + "(\"(yes|no)\"|'(yes|no)'))";
var xmlDecl = @"(?'xmlDecl'<\?xml" + versionInfo + encodingDecl + "?" + sddecl + "?" + s + @"?\?>)"; 
var mixed = @"(?'mixed'\(" + s + @"?\#PCDATA" + "(" + s + @"?\|" + s + "?" + name +")*" + s + "?" + @"\)\*|\(" +s + @"?\#PCDATA" + s + @"?\))";
var children = @"(?'children'unsureifpossible)";
var contentSpec = "(?'contentspec'EMPTY|ANY|"+mixed+"|"+children+")";
var elementDecl = "(?'elementdecl'<!ELEMENT" + s + name + s + contentSpec + s + "?>)";
var stringType = "CDATA";
var tokenizedType = "(ID(REF(S)?)?|ENTIT(Y|IES)|NMTOKENS?)";
var notationType = "(?'notation'NOTATION" +s + @"\(" + s + "?" + name + "(" + s + @"?\|" + s + "?" + name + ")*" + s + @"?\))";
var enumeration = @"(?'enumeration'\(" + s + "?" + nmtoken + "(" + s + @"?\|" + s + "?" + nmtoken + ")*" + s + @"?\))";
var enumeratedType = "(?'enumType'" + notationType + "|" + enumeration +")";
var attType = "(?'attType'" + stringType + "|" + tokenizedType + "|" + enumeratedType + ")";
var attValue = "(?'attValue'\"([^<&\"]|" +reference+ ")*\"|'([^<&']|" + reference + ")*')";
var defaultDecl = @"(?'defaultDecl'\#REQUIRED|\#IMPLIED|(\#FIXED"+s+")?" + attValue + ")";
var attDef = "(?'attDef'"+ s + name + s + attType + s + defaultDecl + ")";
var attListDecl = "(?'attlist'<!ATTLIST" + s + name + attDef + "*" + s + "?>)";
var systemLiteral = "(?'systemLiteral'\"[^\"]*\"|'[^']*')";
var pubIdChar = @"[a-zA-Z0-9'()+,./:=?;!*#@$_%\u0020\u000d\u000a-]";
var pubidLiteral = "(?'pubIdLiteral'\"" + pubIdChar + "*\"|'((?!')" + pubIdChar + ")*')";
var externalID = "(?'externalID'SYSTEM" + s + systemLiteral +"|PUBLIC" + s + pubidLiteral + s + systemLiteral+")";
var nDataDecl = "(?'ndatadecl'"+s + "NDATA" + s + name + ")";
var entityDef  = "(?'entityDef'" + entityValue + "|(" +externalID + nDataDecl + "?))";
var peDef = "(?'pedef'" + entityValue + "|"  + externalID + ")";
var GEDecl = "(?'gedecl'<!ENTITY" + s + name + s + entityDef + s + "?>)";
var PEDecl = "(?'gedecl'<!ENTITY" + s + "%" + s + name + s + peDef + s + "?>)";
var entityDecl = "(?'entityDecl'"+ GEDecl + "|" + PEDecl +")";
var publicID = "(?'publicID'PUBLIC" + s + pubidLiteral + ")";
var notationDecl = "(?'notationDecl'<!NOTATION" +  s + name + s + "(" + externalID + "|" + publicID + ")" + s + "?>)";
var markupDecl = "(?'markupdecl'" + elementDecl + "|" + attListDecl + "|" + entityDecl + "|" + notationDecl + "|" + PI + "|" + comment + ")";
var DeclSep = "(?'declSep'" + pereference + "|" + s + ")";
var intSubSet = @"(?'intSubSet'(" + markupDecl + "|" + DeclSep + ")*)"; 
var docTypeDecl = "(?'doctypedecl'<!DOCTYPE" + s + name + "(" + s + externalID+ ")?" + s + @"?(\[" + intSubSet + @"\]" + s + "?)?>)"; 
var prolog = xmlDecl + "?" + misc + "*(" + docTypeDecl + misc + "*)?"; 
var attribute = "(?'attribute'" +name + eq + attValue + ")";
var CDSect = @"(?'CDSect'<!\[CDATA\[((?!\]\]>)"+c+@")*\]\]>)";
var charData = @"(((?!\]\]>)[^<&])*)";
var content = @"(?>" + // minor optimization... don't backtrack over this (makes failing faster)
		@"<(?'openclose'" + name + @")(" + s + attribute + ")*" + s + @"?/>|"+
		@"<(?'open'"+ name +@")(" + s + attribute + ")*" + s + @"?>|"+
		@"</(?=\k'open'" + s + @"?>)(?'close-open'" + name + ")" + s +@"?>|"
		+reference+@"|"
		+PI+@"|"
		+comment+@"|"
		+CDSect+@"|"
		+charData+@")*" + 
	"(?(open)(?!))";
var rootElement = @"(?'root'(<(?'rootName'" + name + ")(" + s + attribute + ")*" + s + @"?>" + content + @"</\k'rootName'" + s + "?>)|(<(?'rootName'" + name + ")(" + s + attribute + ")*" + s + @"?/>))";
 
var document = "^" + prolog + rootElement + misc + "*" + "$";
 
var testDoc = @"<?xml version='1.0' encoding=""utf-8""?><!DOCTYPE nothtml []><items>
	<item available=""yes"" >
		<name> laptop  </name>
		<![CDATA[something14!$]] 1412]]>
		<"+"\U00010000"+@"quantity>  2 &amp; &#121; &#x234f; </"+"\U00010000"+@"quantity>
	</item><?notxml?>" /* or <?xml?> here */ +@"
	<item available=""yes"" x='' y=""&amp;"">
		<name> mouse </name >
		<quantity> 1 " + /* or ]]> invalid here */  @" </quantity>
	</item>
	<item available=""no"" >
		<!----> <!-- --> <!-- - -->" + /* or <!-- -- --> here */ @"
		<name> keyboad </name>
		<quantity> 0</quantity>
	</item>
</items><!-- stuff can go here --> <!-- yup --> <?pi aasd as!@*&$^!*@&$!@ ?>";
 
//Console.WriteLine(document);
Console.WriteLine(Regex.Match(testDoc, document, RegexOptions.IgnorePatternWhitespace|RegexOptions.Singleline|RegexOptions.ExplicitCapture));
```

---

And here’s the regex (with apologies to [Mail::RFC822::Address](http://www.ex-parrot.com/pdw/Mail-RFC822-Address.html)):

<pre><code style="white-space:nowrap">^(?'localPart'((((\((((?'paren'\()|(?'-paren'\))|([\u0021-\u<br>
0027\u002a-\u005b\u005d-\u007e]|[\u0001-\u0008\u000b\u000c\u<br>
000e-\u001f\u007f])|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)|<br>
\\([\u0021-\u007e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u000b\u000c<br>
\u000e-\u001f\u007f]))*(?(paren)(?!)))\))|([ \t]+((\r\n)[ \t<br>
]+)?|((\r\n)[ \t]+)+))*?(([a-zA-Z0-9!#$%&amp;'*+/=?^_`{|}~-]+)|(<br>
"(([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)?(([\u0021\u0023-\u<br>
005b\u005d-\u007e]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u<br>
007f])|\\([\u0021-\u007e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u000<br>
b\u000c\u000e-\u001f\u007f])))*([ \t]+((\r\n)[ \t]+)?|((\r\n<br>
)[ \t]+)+)?"))((\((((?'paren'\()|(?'-paren'\))|([\u0021-\u00<br>
27\u002a-\u005b\u005d-\u007e]|[\u0001-\u0008\u000b\u000c\u00<br>
0e-\u001f\u007f])|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)|\\<br>
([\u0021-\u007e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u000b\u000c\u<br>
000e-\u001f\u007f]))*(?(paren)(?!)))\))|([ \t]+((\r\n)[ \t]+<br>
)?|((\r\n)[ \t]+)+))*?)(\.(((\((((?'paren'\()|(?'-paren'\))|<br>
([\u0021-\u0027\u002a-\u005b\u005d-\u007e]|[\u0001-\u0008\u0<br>
00b\u000c\u000e-\u001f\u007f])|([ \t]+((\r\n)[ \t]+)?|((\r\n<br>
)[ \t]+)+)|\\([\u0021-\u007e]|[ \t]|[\r\n\0]|[\u0001-\u0008\<br>
u000b\u000c\u000e-\u001f\u007f]))*(?(paren)(?!)))\))|([ \t]+<br>
((\r\n)[ \t]+)?|((\r\n)[ \t]+)+))*?(([a-zA-Z0-9!#$%&amp;'*+/=?^_<br>
`{|}~-]+)|("(([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)?(([\u00<br>
21\u0023-\u005b\u005d-\u007e]|[\u0001-\u0008\u000b\u000c\u00<br>
0e-\u001f\u007f])|\\([\u0021-\u007e]|[ \t]|[\r\n\0]|[\u0001-<br>
\u0008\u000b\u000c\u000e-\u001f\u007f])))*([ \t]+((\r\n)[ \t<br>
]+)?|((\r\n)[ \t]+)+)?"))((\((((?'paren'\()|(?'-paren'\))|([<br>
\u0021-\u0027\u002a-\u005b\u005d-\u007e]|[\u0001-\u0008\u000<br>
b\u000c\u000e-\u001f\u007f])|([ \t]+((\r\n)[ \t]+)?|((\r\n)[<br>
\t]+)+)|\\([\u0021-\u007e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u0<br>
00b\u000c\u000e-\u001f\u007f]))*(?(paren)(?!)))\))|([ \t]+((<br>
\r\n)[ \t]+)?|((\r\n)[ \t]+)+))*?))*))@(?'domain'((((\((((?'<br>
paren'\()|(?'-paren'\))|([\u0021-\u0027\u002a-\u005b\u005d-\<br>
u007e]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f])|([ \t<br>
]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)|\\([\u0021-\u007e]|[ \t]|<br>
[\r\n\0]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f]))*(?<br>
(paren)(?!)))\))|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+))*?(<br>
([a-zA-Z0-9!#$%&amp;'*+/=?^_`{|}~-]+)|("(([ \t]+((\r\n)[ \t]+)?|<br>
((\r\n)[ \t]+)+)?(([\u0021\u0023-\u005b\u005d-\u007e]|[\u000<br>
1-\u0008\u000b\u000c\u000e-\u001f\u007f])|\\([\u0021-\u007e]<br>
|[ \t]|[\r\n\0]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007<br>
f])))*([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)?"))((\((((?'pa<br>
ren'\()|(?'-paren'\))|([\u0021-\u0027\u002a-\u005b\u005d-\u0<br>
07e]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f])|([ \t]+<br>
((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)|\\([\u0021-\u007e]|[ \t]|[\<br>
r\n\0]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f]))*(?(p<br>
aren)(?!)))\))|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+))*?)(\<br>
.(((\((((?'paren'\()|(?'-paren'\))|([\u0021-\u0027\u002a-\u0<br>
05b\u005d-\u007e]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u0<br>
07f])|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)|\\([\u0021-\u0<br>
07e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\<br>
u007f]))*(?(paren)(?!)))\))|([ \t]+((\r\n)[ \t]+)?|((\r\n)[<br>
\t]+)+))*?(([a-zA-Z0-9!#$%&amp;'*+/=?^_`{|}~-]+)|("(([ \t]+((\r\<br>
n)[ \t]+)?|((\r\n)[ \t]+)+)?(([\u0021\u0023-\u005b\u005d-\u0<br>
07e]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f])|\\([\u0<br>
021-\u007e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u000b\u000c\u000e-<br>
\u001f\u007f])))*([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)?"))<br>
((\((((?'paren'\()|(?'-paren'\))|([\u0021-\u0027\u002a-\u005<br>
b\u005d-\u007e]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007<br>
f])|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)|\\([\u0021-\u007<br>
e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u0<br>
07f]))*(?(paren)(?!)))\))|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t<br>
]+)+))*?))*)|(((\((((?'paren'\()|(?'-paren'\))|([\u0021-\u00<br>
27\u002a-\u005b\u005d-\u007e]|[\u0001-\u0008\u000b\u000c\u00<br>
0e-\u001f\u007f])|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)|\\<br>
([\u0021-\u007e]|[ \t]|[\r\n\0]|[\u0001-\u0008\u000b\u000c\u<br>
000e-\u001f\u007f]))*(?(paren)(?!)))\))|([ \t]+((\r\n)[ \t]+<br>
)?|((\r\n)[ \t]+)+))*?\[(([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]<br>
+)+)?([!-Z^-~]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f<br>
]))*([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+)?\]((\((((?'paren<br>
'\()|(?'-paren'\))|([\u0021-\u0027\u002a-\u005b\u005d-\u007e<br>
]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f])|([ \t]+((\<br>
r\n)[ \t]+)?|((\r\n)[ \t]+)+)|\\([\u0021-\u007e]|[ \t]|[\r\n<br>
\0]|[\u0001-\u0008\u000b\u000c\u000e-\u001f\u007f]))*(?(pare<br>
n)(?!)))\))|([ \t]+((\r\n)[ \t]+)?|((\r\n)[ \t]+)+))*?))$</code></pre>