# gmxapi_graph_0_2
Specification for second generation gmxapi work serialization.

RST documentation in `docs`

Python reference implementation in `serialization`. Assumes Python version >= 3.5.

Jupyter notebooks in `notebooks`.

Test code for Python package in `test`. Run with pytest.

## Documentation

Builds with Sphinx and Plantuml.

    pip install sphinxcontrib-plantuml
    sphinx-build -b html -c docs docs build/html
    open build/html/index.html

Assumes plantuml is installed and that a suitable wrapper script exists at
`/usr/bin/plantuml` as described at
https://pypi.org/project/sphinxcontrib-plantuml/.
Note that a Linux distribution (Ubuntu, for instance) may package plantuml with
a suitable wrapper script at `/usr/bin/plantuml`.
If not available, consider copying the script from
https://pypi.org/project/sphinxcontrib-plantuml/
and change `docs/conf.py` to refer to the correct plantuml wrapper location.
