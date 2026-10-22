<h1 align="center">EMMET-CSS</h1>

Emmet también funciona para CSS, funciona prácticamente igual que en HTML.

En la mayoría de casos en CSS separa la propiedad y el valor mediante el uso de dos puntos (`:`)

-Ejemplos

```CSS
/*Es la misma forma para distintos los Selectores,Atributos,Valores   */
selector{
    /* Emmet --> Resultado */
    m0 --> margin:0;
    m:a --> margin: auto;
    pt --> padding: top;
    p2rem-->  padding: 2rem;
    w50% --> width: 50%;
    bxz --> box-sizing:border-box;
    pos:r --> position:relative;
    m0-auto -->  margin: 0 auto; /*Usar operador -  para insertar espacio entre valores*/
    fs:italic --> font-style: italic;
    fz:2rem --> font-size: 2rem;
}

/*Es la misma forma para distintos los Selectores,Atributos,Valores   */
selector{
    /* Emmet --> w50%+h700px+m:0-a+p2rem */
    /* Resultado: */
    width: 50%;
    height: 700px;
    margin: 0 auto;
    padding: 2rem;
}

```

> [!NOTE]
> 
> No perder el tiempo aprendiendo todos, existen muchísimos atajos, consultar más atajos en [cheat-sheet](https://docs.emmet.io/cheat-sheet/). Los que más utilices los recordaras de tanto usarlos.
