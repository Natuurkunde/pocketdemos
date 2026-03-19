In conversion md to typst, things go wrong. For instance in subfigures: 

:::markdown
````{figure}
:label: fig_experiment_free_fall

```{figure} ../images/20250513_090352.*
:width: 50%
:label: fig_352

Kun je je hand versnellen met meer dan g?
```

```{figure} ../images/20250513_090908.jpg
:width: 50%
:label: fig_908

Kun je een vel papier net zo snel laten vallen als een boek? 
```

Vallen met $g$, langzamer of sneller?
```` 
::: 

becomes:

#show figure: set block(breakable: false)
#subpar.grid(figure(
image("files/20250513_091205-a7302a264bf4beb5257fdce51351d53b.jpg", width: 60%)
, caption: []), <fig_205>,
figure(
image("files/20250513_091700-811688574a4f6237b00b72640af5dd8c.jpg", width: 60%)
, caption: []), <fig_700>,
columns: 2,
label: <bakjesval>,
  caption: [
Zou luchtweerstand evenredig zijn met $v^2$ in plaats van $v$? Als dat zo zou zijn, dan komen de bakjes in de figuur niet tegelijk op de grond.
],
  kind: "figure",
  supplement: [Figure],
)

It then doesn't follow the template where for instance I style numbering with a prefix of the chapter number: 
```{code-block} typ
  set figure(numbering: (..args) => {
    let chapter = counter(heading).display((..nums) => nums.pos().at(0))
    [#chapter.#numbering("1", ..args.pos())]
  })
```

so rather than: 
```{code-block} typ
#subpar.grid(
  figure(...),
  figure(...),
  columns: 2,
  label: <bakjesval>,
  caption: [...],
  kind: "figure",
  supplement: [Figure],
)
```

it should become: 
```
#figure(
  subpar.grid(
    image("files/20250513_091205-a7302a264bf4beb5257fdce51351d53b.jpg", width: 60%),
    image("files/20250513_091700-811688574a4f6237b00b72640af5dd8c.jpg", width: 60%),
    columns: 2,
  ),
  caption: [
    Zou luchtweerstand evenredig zijn met $v^2$ in plaats van $v$? Als dat zo zou zijn, dan komen de bakjes in de figuur niet tegelijk op de grond.
  ],
  kind: "figure",
  supplement: [Figure],
) <bakjesval>
```

replace in: https://github.com/jupyter-book/mystmd/blob/main/packages/myst-to-typst/src/container.ts

--beginning of code snippet--

```typescript
if (nonCaptions && nonCaptions.length > 1) {
  const allSubFigs =
    nonCaptions.filter((item: GenericNode) => item.type === 'container').length ===
    nonCaptions.length;
  state.useMacro('#import "@preview/subpar:0.2.2"');
  state.useMacro('#let breakableDefault = true');
  state.write(
    `#show figure: set block(breakable: ${allSubFigs ? 'false' : 'breakableDefault'})\n`,
  );
  state.write('#subpar.grid(');
  let columns = nonCaptions.length <= 3 ? nonCaptions.length : 2; // TODO: allow this to be customized
  nonCaptions.forEach((item: GenericNode) => {
    if (item.type === 'container') {
      state.write('figure(\n');
      state.renderChildren(item);
      state.write('\n, caption: []),'); // TODO: add sub-captions
      if (item.identifier) {
        state.write(` <${item.identifier}>,`);
      }
      state.write('\n');
    } else {
      renderFigureChild(item, state);
      state.write(',\n');
      columns = 1;
    }
  });
  state.write(`columns: ${columns},\n`);
  if (label) {
    state.write(`label: <${label}>,`);
    label = undefined;
  }
}
```
--end of code snippet--
with
--beginning of code snippet--

```typescript
if (nonCaptions && nonCaptions.length > 1) {
  const allSubFigs =
    nonCaptions.filter((item: GenericNode) => item.type === 'container').length ===
    nonCaptions.length;

  state.useMacro('#import "@preview/subpar:0.2.2"');
  state.useMacro('#let breakableDefault = true');
  state.write(
    `#show figure: set block(breakable: ${allSubFigs ? 'false' : 'breakableDefault'})\n`,
  );

  // Outer real figure
  state.write('#figure([\n');
  state.write('#subpar.grid(\n');

  let columns = nonCaptions.length <= 3 ? nonCaptions.length : 2;

  nonCaptions.forEach((item: GenericNode) => {
    if (item.type === 'container') {
      // Render container contents directly, not as inner figure(...)
      // so only the outer figure gets numbering/references.
      if (item.children && item.children.length === 1) {
        renderFigureChild(item.children[0], state);
      } else {
        renderFigureChild(item, state);
      }
      if (item.identifier) {
        state.write(` <${item.identifier}>`);
      }
      state.write(',\n');
    } else {
      renderFigureChild(item, state);
      state.write(',\n');
      columns = 1;
    }
  });

  state.write(`columns: ${columns},\n`);
  state.write(')\n');
  state.write('],');
}
```
--end of code snippet--