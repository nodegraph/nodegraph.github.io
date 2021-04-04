---
layout: default
title: Dot Node
parent: YouMacro Node Graph
nav_order: 7
---

# Dot Node
{: .no_toc }

A dot node simply passes its input value to its output value. It does not perform any computation or modification of data. It's main purpose is help organize the wires and lines in the node graph. For example, it is often cleaner to avoid wires overlapping each other in the node graph. Dot nodes can be used to guide wires and links around areas of high congestion. 

Dot node base class header.

{% highlight cpp %}
class COMPUTES_EXPORT DotNodeCompute: public Compute {
 public:
  COMPONENT_ID(Compute, DotNodeCompute);
  DotNodeCompute(Entity* entity);
  virtual ~DotNodeCompute();

  // Input and Outputs.
  virtual void create_inputs_outputs(const EntityConfig& config = EntityConfig());

  // Our hints.
  static QJsonObject init_hints();
  static const QJsonObject _hints;
  virtual const QJsonObject& get_hints() const {return _hints;}

protected:
  // Our state.
  virtual bool update_state();
};

{% endhighlight %}

Dot node base class implementation.

{% highlight cpp %}
DotNodeCompute::DotNodeCompute(Entity* entity)
    : Compute(entity, kDID()) {
}

DotNodeCompute::~DotNodeCompute() {
}

void DotNodeCompute::create_inputs_outputs(const EntityConfig& config) {
  external();
  Compute::create_inputs_outputs(config);
  create_main_input(config);
  create_main_output(config);
  // The type of the input and output is set to be an object.
  // This allows us to connect our input and output to any plug types,
  // as the input computes will convert values automatically.
}

const QJsonObject DotNodeCompute::_hints = DotNodeCompute::init_hints();
QJsonObject DotNodeCompute::init_hints() {
  QJsonObject m;
  add_main_input_hint(m);
  return m;
}

bool DotNodeCompute::update_state() {
  internal();
  Compute::update_state();
  set_main_output(_inputs->get_main_input_object());
  return true;
}
{% endhighlight %}

