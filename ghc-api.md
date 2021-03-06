# Developments in the GHC API



This page summarises changes to the GHC API that will make it easier for
tool writers to use the GHC API.


1. More parser entrypoints, to explicitly parse fragments (Andrew Gibiansky)

>
>
> The following parsers are now provided
>
>

>
> >
> >
> > `parseModule`, `parseImport`, `parseStatement`,  `parseDeclaration`,
> > `parseExpression`, `parseTypeSignature`,  `parseFullStmt`, `parseStmt`,
> > `parseIdentifier`,  `parseType`, `parseHeader`
> >
> >
>

1. No more landmines in the AST (Alan Zimmerman).

>
>
> In the past it was difficult to work with the GHC AST as any generic traversals had to carefully tiptoe around an assortment of panic and undefined expressions. These have now been removed, by uses of `PostRn`, `PostTc`, etc, allowing standard traversal libraries to be used.  See module `hsSyn/PlaceHolder`.
>
>

>
>
> The relevant commit is `7d3f2dfc7a45d741224c521e0f2a616a89f9506f`
>
>

1. Introduce an annotation structure to the `ParsedSource` to record the location of un-captured keywords (Alan Zimmerman).  

>
>
> At the moment the location of let / in / if / then / else / do etc is not captured in the AST. This makes it difficult to parse some source, transform the AST, and then output it again preserving the original layout.
>
>

>
>
> Wiki page: [ApiAnnotations](api-annotations)
>
>

