.. role:: hidden
    :class: hidden-section

.. currentmodule:: {{ module }}

{% if objname in ["FastGelu", "GatherV2", "TensorAdd", "Gelu"] %}
{{ fullname | underline }}

.. autofunction:: {{ fullname }}
{% elif objname[0].istitle() %}
{{ fullname | underline }}

.. autoclass:: {{ name }}
    :members:

{% else %}
{{ fullname | underline }}

.. autofunction:: {{ fullname }}
{% endif %}

..
  autogenerated from _templates/classtemplate.rst
  note it does not have :inherited-members:
