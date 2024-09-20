---
title: So, it turns out that .NET’s Regex are more powerful t̖̱̍ͭ͊h̟ͨͨa̞̖̙̔̇n͇̝͚̤̒́ͨ̐ ̯͖̏̌̔Ị̟̮̱̥̇̐̎͂ͬ͗̒ ̪̹̱͙̘ͦ̉ͪͪͣ̉͊o͕̥̝͇͙ͪ͊ͤ̑̂̽́r͔̭̪̮̟͗̍ͨ͗͛ͣḭ̝̜͈ͫ́g̥̹̥̜̦̓̇̓i̪͕̭̞͛ͯ̓͛̔̾ͫn̘̗a̰̜ͨͪ͊l̩͑̐̐́ͥ̚l̜ͨ͋̈ẙͦ́ ̟̬̬̫͙̤ͭ̚t̳͎̱̗̲́h͔͙̰̬̊̈́͊̾o͉ͫ̌̄u͉̲̥g̏ͥ̑̅̽̇h̻͇̥̰̯ͥͯṱ̯̏̄̒͒ͫ̃.͖̟͍̘̼̼̍̐̀͊̓́…
date: 2011-05-10
tags:
  - software-horrors
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

<pre><code style="white-space:nowrap">^(?'xmlDecl'&lt;\?xml(?'versionInfo'([\u0020\u0009\u000d\u000a]+)version(?'eq'([\u0<br>
020\u0009\u000d\u000a]+)?=([\u0020\u0009\u000d\u000a]+)?)('1\.[0-9]+'|"1\.[0-9]+<br>
"))(?'encodingDecl'([\u0020\u0009\u000d\u000a]+)encoding(?'eq'([\u0020\u0009\u00<br>
0d\u000a]+)?=([\u0020\u0009\u000d\u000a]+)?)("(?'encName'[A-Za-z][A-Za-z0-9._-]*<br>
)"|'(?'encName'[A-Za-z][A-Za-z0-9._-]*)'))?(?'sddecl'([\u0020\u0009\u000d\u000a]<br>
+)standalone(?'eq'([\u0020\u0009\u000d\u000a]+)?=([\u0020\u0009\u000d\u000a]+)?)<br>
("(yes|no)"|'(yes|no)'))?([\u0020\u0009\u000d\u000a]+)?\?&gt;)?(?'misc'(?'comment'&lt;<br>
!--((?!--)([\u0009\u000a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\udbff][\udc0<br>
0-\udfff])))*--&gt;)|(?'PI'&lt;\?(?'pitarget'(?![xX][mM][lL])(?'name'([:A-Z_a-z\u00C0-<br>
\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u<br>
218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc0<br>
0-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F<br>
-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\<br>
uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040]<br>
)*))(([\u0020\u0009\u000d\u000a]+)((?!\?&gt;)([\u0009\u000a\u000d\u0020-\ud7ff\ue00<br>
0-\ufffd]|([\ud800-\udbff][\udc00-\udfff])))*)?\?&gt;)|([\u0020\u0009\u000d\u000a]+<br>
))*((?'doctypedecl'&lt;!DOCTYPE([\u0020\u0009\u000d\u000a]+)(?'name'([:A-Z_a-z\u00C<br>
0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-<br>
\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\ud<br>
c00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u03<br>
7F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0<br>
-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u204<br>
0])*)(([\u0020\u0009\u000d\u000a]+)(?'externalID'SYSTEM([\u0020\u0009\u000d\u000<br>
a]+)(?'systemLiteral'"[^"]*"|'[^']*')|PUBLIC([\u0020\u0009\u000d\u000a]+)(?'pubI<br>
dLiteral'"[a-zA-Z0-9'()+,./:=?;!*#@$_%\u0020\u000d\u000a-]*"|'((?!')[a-zA-Z0-9'(<br>
)+,./:=?;!*#@$_%\u0020\u000d\u000a-])*')([\u0020\u0009\u000d\u000a]+)(?'systemLi<br>
teral'"[^"]*"|'[^']*')))?([\u0020\u0009\u000d\u000a]+)?(\[(?'intSubSet'((?'marku<br>
pdecl'(?'elementdecl'&lt;!ELEMENT([\u0020\u0009\u000d\u000a]+)(?'name'([:A-Z_a-z\u0<br>
0C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u207<br>
0-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\<br>
udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u<br>
037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFD<br>
F0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2<br>
040])*)([\u0020\u0009\u000d\u000a]+)(?'contentspec'EMPTY|ANY|(?'mixed'\(([\u0020<br>
\u0009\u000d\u000a]+)?\#PCDATA(([\u0020\u0009\u000d\u000a]+)?\|([\u0020\u0009\u0<br>
00d\u000a]+)?(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u0<br>
37D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDC<br>
F\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-<br>
\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u<br>
2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[<br>
-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*))*([\u0020\u0009\u000d\u000a]+)?\)\*|\(<br>
([\u0020\u0009\u000d\u000a]+)?\#PCDATA([\u0020\u0009\u000d\u000a]+)?\))|(?'child<br>
ren'unsureifpossible))([\u0020\u0009\u000d\u000a]+)?&gt;)|(?'attlist'&lt;!ATTLIST([\u0<br>
020\u0009\u000d\u000a]+)(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02F<br>
F\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\<br>
uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u<br>
00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u21<br>
8F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-<br>
\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*)(?'attDef'([\u0020\u0009\u00<br>
0d\u000a]+)(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037<br>
D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\<br>
uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u<br>
00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2F<br>
EF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.<br>
0-9\u00B7\u0300-\u036F\u203F-\u2040])*)([\u0020\u0009\u000d\u000a]+)(?'attType'C<br>
DATA|(ID(REF(S)?)?|ENTIT(Y|IES)|NMTOKENS?)|(?'enumType'(?'notation'NOTATION([\u0<br>
020\u0009\u000d\u000a]+)\(([\u0020\u0009\u000d\u000a]+)?(?'name'([:A-Z_a-z\u00C0<br>
-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\<br>
u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc<br>
00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037<br>
F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-<br>
\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040<br>
])*)(([\u0020\u0009\u000d\u000a]+)?\|([\u0020\u0009\u000d\u000a]+)?(?'name'([:A-<br>
Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u2<br>
00D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\<br>
udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-<br>
\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\u<br>
FDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u<br>
203F-\u2040])*))*([\u0020\u0009\u000d\u000a]+)?\))|(?'enumeration'\(([\u0020\u00<br>
09\u000d\u000a]+)?(?'nmtoken'(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\<br>
u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF<br>
900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u<br>
036F\u203F-\u2040])+)(([\u0020\u0009\u000d\u000a]+)?\|([\u0020\u0009\u000d\u000a<br>
]+)?(?'nmtoken'(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u<br>
037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFD<br>
F0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2<br>
040])+))*([\u0020\u0009\u000d\u000a]+)?\))))([\u0020\u0009\u000d\u000a]+)(?'defa<br>
ultDecl'\#REQUIRED|\#IMPLIED|(\#FIXED([\u0020\u0009\u000d\u000a]+))?(?'attValue'<br>
"([^&lt;&amp;"]|(?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6<br>
\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u<br>
3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_<br>
a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200<br>
D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\ud<br>
bff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|<br>
x[0-9a-fA-F]+);()))*"|'([^&lt;&amp;']|(?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00<br>
C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070<br>
-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\u<br>
dc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u0<br>
37F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF<br>
0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u20<br>
40])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*')))*([\u0020\u0009\u000d\u000a]+)?&gt;)|<br>
(?'entityDecl'(?'gedecl'&lt;!ENTITY([\u0020\u0009\u000d\u000a]+)(?'name'([:A-Z_a-z\<br>
u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2<br>
070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff]<br>
[\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D<br>
\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\u<br>
FDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\<br>
u2040])*)([\u0020\u0009\u000d\u000a]+)(?'entityDef'(?'entityValue'"([^%&amp;"]|%(?'n<br>
ame'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\<br>
u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|(<br>
[\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02<br>
FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF<br>
\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300<br>
-\u036F\u203F-\u2040])*);|(?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u<br>
00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u21<br>
8F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-<br>
\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\<br>
u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uF<br>
FFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*<br>
);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*"|'([^%&amp;']|%(?'name'([:A-Z_a-z\u00C0-\u00D6<br>
\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u<br>
2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udf<br>
ff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FF<br>
F\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]<br>
|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);|(<br>
?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u0<br>
2FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7F<br>
F\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-<br>
\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u<br>
218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc0<br>
0-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-<br>
F]+);()))*')|((?'externalID'SYSTEM([\u0020\u0009\u000d\u000a]+)(?'systemLiteral'<br>
"[^"]*"|'[^']*')|PUBLIC([\u0020\u0009\u000d\u000a]+)(?'pubIdLiteral'"[a-zA-Z0-9'<br>
()+,./:=?;!*#@$_%\u0020\u000d\u000a-]*"|'((?!')[a-zA-Z0-9'()+,./:=?;!*#@$_%\u002<br>
0\u000d\u000a-])*')([\u0020\u0009\u000d\u000a]+)(?'systemLiteral'"[^"]*"|'[^']*'<br>
))(?'ndatadecl'([\u0020\u0009\u000d\u000a]+)NDATA([\u0020\u0009\u000d\u000a]+)(?<br>
'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FF<br>
F\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]<br>
|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u<br>
02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7<br>
FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u03<br>
00-\u036F\u203F-\u2040])*))?))([\u0020\u0009\u000d\u000a]+)?&gt;)|(?'gedecl'&lt;!ENTIT<br>
Y([\u0020\u0009\u000d\u000a]+)%([\u0020\u0009\u000d\u000a]+)(?'name'([:A-Z_a-z\u<br>
00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u20<br>
70-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][<br>
\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\<br>
u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uF<br>
DF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u<br>
2040])*)([\u0020\u0009\u000d\u000a]+)(?'pedef'(?'entityValue'"([^%&amp;"]|%(?'name'(<br>
[:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C<br>
-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud8<br>
00-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0<br>
370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF90<br>
0-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u03<br>
6F\u203F-\u2040])*);|(?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\<br>
u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2<br>
C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udff<br>
f]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF<br>
\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|<br>
([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())<br>
|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*"|'([^%&amp;']|%(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D<br>
8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-<br>
\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))<br>
(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u20<br>
0C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\u<br>
d800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);|(?'ref<br>
erence'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u<br>
0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF9<br>
00-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D<br>
6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\<br>
u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\ud<br>
fff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);<br>
()))*')|(?'externalID'SYSTEM([\u0020\u0009\u000d\u000a]+)(?'systemLiteral'"[^"]*<br>
"|'[^']*')|PUBLIC([\u0020\u0009\u000d\u000a]+)(?'pubIdLiteral'"[a-zA-Z0-9'()+,./<br>
:=?;!*#@$_%\u0020\u000d\u000a-]*"|'((?!')[a-zA-Z0-9'()+,./:=?;!*#@$_%\u0020\u000<br>
d\u000a-])*')([\u0020\u0009\u000d\u000a]+)(?'systemLiteral'"[^"]*"|'[^']*')))([\<br>
u0020\u0009\u000d\u000a]+)?&gt;))|(?'notationDecl'&lt;!NOTATION([\u0020\u0009\u000d\u0<br>
00a]+)(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u03<br>
7F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0<br>
-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\<br>
u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3<br>
001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u<br>
00B7\u0300-\u036F\u203F-\u2040])*)([\u0020\u0009\u000d\u000a]+)((?'externalID'SY<br>
STEM([\u0020\u0009\u000d\u000a]+)(?'systemLiteral'"[^"]*"|'[^']*')|PUBLIC([\u002<br>
0\u0009\u000d\u000a]+)(?'pubIdLiteral'"[a-zA-Z0-9'()+,./:=?;!*#@$_%\u0020\u000d\<br>
u000a-]*"|'((?!')[a-zA-Z0-9'()+,./:=?;!*#@$_%\u0020\u000d\u000a-])*')([\u0020\u0<br>
009\u000d\u000a]+)(?'systemLiteral'"[^"]*"|'[^']*'))|(?'publicID'PUBLIC([\u0020\<br>
u0009\u000d\u000a]+)(?'pubIdLiteral'"[a-zA-Z0-9'()+,./:=?;!*#@$_%\u0020\u000d\u0<br>
00a-]*"|'((?!')[a-zA-Z0-9'()+,./:=?;!*#@$_%\u0020\u000d\u000a-])*')))([\u0020\u0<br>
009\u000d\u000a]+)?&gt;)|(?'PI'&lt;\?(?'pitarget'(?![xX][mM][lL])(?'name'([:A-Z_a-z\u0<br>
0C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u207<br>
0-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\<br>
udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u<br>
037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFD<br>
F0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2<br>
040])*))(([\u0020\u0009\u000d\u000a]+)((?!\?&gt;)([\u0009\u000a\u000d\u0020-\ud7ff\<br>
ue000-\ufffd]|([\ud800-\udbff][\udc00-\udfff])))*)?\?&gt;)|(?'comment'&lt;!--((?!--)([<br>
\u0009\u000a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\udbff][\udc00-\udfff])))<br>
*--&gt;))|(?'declSep'%(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u03<br>
70-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900<br>
-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\<br>
u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2<br>
C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udff<br>
f]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);|([\u0020\u0009\u000d\u000a]+)))<br>
*)\]([\u0020\u0009\u000d\u000a]+)?)?&gt;)(?'misc'(?'comment'&lt;!--((?!--)([\u0009\u00<br>
0a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\udbff][\udc00-\udfff])))*--&gt;)|(?'P<br>
I'&lt;\?(?'pitarget'(?![xX][mM][lL])(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u0<br>
0F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u300<br>
1-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z<br>
\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u<br>
2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff<br>
][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*))(([\u0020\u0009\u0<br>
00d\u000a]+)((?!\?&gt;)([\u0009\u000a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\ud<br>
bff][\udc00-\udfff])))*)?\?&gt;)|([\u0020\u0009\u000d\u000a]+))*)?(?'root'(&lt;(?'root<br>
Name'(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037<br>
F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-<br>
\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u<br>
00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u30<br>
01-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u0<br>
0B7\u0300-\u036F\u203F-\u2040])*))(([\u0020\u0009\u000d\u000a]+)(?'attribute'(?'<br>
name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF<br>
\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|<br>
([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u0<br>
2FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7F<br>
F\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u030<br>
0-\u036F\u203F-\u2040])*)(?'eq'([\u0020\u0009\u000d\u000a]+)?=([\u0020\u0009\u00<br>
0d\u000a]+)?)(?'attValue'"([^&lt;&amp;"]|(?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\<br>
u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2<br>
070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff]<br>
[\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D<br>
\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\u<br>
FDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\<br>
u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*"|'([^&lt;&amp;']|(?'reference'(?'entityRe<br>
f'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-<br>
\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\u<br>
FFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00<br>
F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001<br>
-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B<br>
7\u0300-\u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*')))*([\u0020<br>
\u0009\u000d\u000a]+)?&gt;(?&gt;&lt;(?'openclose'(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u<br>
00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2F<br>
EF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:<br>
A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\<br>
u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800<br>
-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*))(([\u0020\u<br>
0009\u000d\u000a]+)(?'attribute'(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00<br>
F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001<br>
-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\<br>
u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2<br>
070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff]<br>
[\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*)(?'eq'([\u0020\u0009<br>
\u000d\u000a]+)?=([\u0020\u0009\u000d\u000a]+)?)(?'attValue'"([^&lt;&amp;"]|(?'referenc<br>
e'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-<br>
\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\u<br>
FDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00<br>
D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00<br>
-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff])<br>
)|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*<br>
"|'([^&lt;&amp;']|(?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00<br>
F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF<br>
\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-<br>
Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u2<br>
00D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\<br>
udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())|&amp;\#([0-9]<br>
+|x[0-9a-fA-F]+);()))*')))*([\u0020\u0009\u000d\u000a]+)?/&gt;|&lt;(?'open'(?'name'([:<br>
A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\<br>
u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800<br>
-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u037<br>
0-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-<br>
\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F<br>
\u203F-\u2040])*))(([\u0020\u0009\u000d\u000a]+)(?'attribute'(?'name'([:A-Z_a-z\<br>
u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2<br>
070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff]<br>
[\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D<br>
\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\u<br>
FDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\<br>
u2040])*)(?'eq'([\u0020\u0009\u000d\u000a]+)?=([\u0020\u0009\u000d\u000a]+)?)(?'<br>
attValue'"([^&lt;&amp;"]|(?'reference'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00<br>
D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00<br>
-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff])<br>
)(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u2<br>
00C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\<br>
ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())|&amp;\<br>
#([0-9]+|x[0-9a-fA-F]+);()))*"|'([^&lt;&amp;']|(?'reference'(?'entityRef'&amp;(?'name'([:A-<br>
Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u2<br>
00D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\<br>
udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-<br>
\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\u<br>
FDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u<br>
203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*')))*([\u0020\u0009\u000d\u00<br>
0a]+)?&gt;|&lt;/(?=\k'open'([\u0020\u0009\u000d\u000a]+)?&gt;)(?'close-open'(?'name'([:A-<br>
Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u2<br>
00D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\<br>
udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-<br>
\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\u<br>
FDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u<br>
203F-\u2040])*))([\u0020\u0009\u000d\u000a]+)?&gt;|(?'reference'(?'entityRef'&amp;(?'na<br>
me'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u<br>
200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([<br>
\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02F<br>
F\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\<br>
uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-<br>
\u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);())|(?'PI'&lt;\?(?'pitarget'(<br>
?![xX][mM][lL])(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\<br>
u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uF<br>
DCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D<br>
8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-<br>
\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))<br>
|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*))(([\u0020\u0009\u000d\u000a]+)((?!\?<br>
&gt;)([\u0009\u000a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\udbff][\udc00-\udfff<br>
])))*)?\?&gt;)|(?'comment'&lt;!--((?!--)([\u0009\u000a\u000d\u0020-\ud7ff\ue000-\ufffd<br>
]|([\ud800-\udbff][\udc00-\udfff])))*--&gt;)|(?'CDSect'&lt;!\[CDATA\[((?!\]\]&gt;)([\u000<br>
9\u000a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\udbff][\udc00-\udfff])))*\]\]<br>
&gt;)|(((?!\]\]&gt;)[^&lt;&amp;])*))*(?(open)(?!))&lt;/\k'rootName'([\u0020\u0009\u000d\u000a]+)<br>
?&gt;)|(&lt;(?'rootName'(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u037<br>
0-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-<br>
\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u<br>
00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C<br>
00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff<br>
]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*))(([\u0020\u0009\u000d\u000a]+)(?'<br>
attribute'(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D<br>
\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\u<br>
FDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u0<br>
0F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FE<br>
F\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0<br>
-9\u00B7\u0300-\u036F\u203F-\u2040])*)(?'eq'([\u0020\u0009\u000d\u000a]+)?=([\u0<br>
020\u0009\u000d\u000a]+)?)(?'attValue'"([^&lt;&amp;"]|(?'reference'(?'entityRef'&amp;(?'nam<br>
e'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u2<br>
00C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\<br>
ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF<br>
\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\u<br>
F900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\<br>
u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*"|'([^&lt;&amp;']|(?'referenc<br>
e'(?'entityRef'&amp;(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-<br>
\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\u<br>
FDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-z\u00C0-\u00D6\u00<br>
D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00<br>
-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff])<br>
)|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*);())|&amp;\#([0-9]+|x[0-9a-fA-F]+);()))*<br>
')))*([\u0020\u0009\u000d\u000a]+)?/&gt;))(?'misc'(?'comment'&lt;!--((?!--)([\u0009\u0<br>
00a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\udbff][\udc00-\udfff])))*--&gt;)|(?'<br>
PI'&lt;\?(?'pitarget'(?![xX][mM][lL])(?'name'([:A-Z_a-z\u00C0-\u00D6\u00D8-\u00F6\u<br>
00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\u2070-\u218F\u2C00-\u2FEF\u30<br>
01-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbff][\udc00-\udfff]))(([:A-Z_a-<br>
z\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u02FF\u0370-\u037D\u037F-\u1FFF\u200C-\u200D\<br>
u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD]|([\ud800-\udbf<br>
f][\udc00-\udfff]))|[-.0-9\u00B7\u0300-\u036F\u203F-\u2040])*))(([\u0020\u0009\u<br>
000d\u000a]+)((?!\?&gt;)([\u0009\u000a\u000d\u0020-\ud7ff\ue000-\ufffd]|([\ud800-\u<br>
dbff][\udc00-\udfff])))*)?\?&gt;)|([\u0020\u0009\u000d\u000a]+))*$</code></pre>