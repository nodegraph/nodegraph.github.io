---
layout: default
title: Input
parent: YouMacro Node Graph
nav_order: 2
---

# Input
{: .no_toc }

The Input base class calculates the current input value for a given input to a node. Often nodes have many inputs and many outputs. Parameters on a node are considered to be inputs to a node.

Input base class header.

{% highlight cpp %}
class COMPUTES_EXPORT InputCompute: public Compute  {
 public:
  COMPONENT_ID(Compute, InputCompute);
  InputCompute(Entity* entity);
  virtual ~InputCompute();

  // We override to stop creating the inputs and outputs namespace.
  virtual void create_inputs_outputs(const EntityConfig& config = EntityConfig()) {}

  // The unconnected value.
  void set_unconnected_value(const QJsonValue& value);
  const QJsonValue& get_unconnected_value() const;

  // Exposure.
  void set_exposed(bool exposed);
  bool is_exposed() const;

  // Our dynamic output compute dependency.
  bool can_link_output_compute(const Dep<OutputCompute>& output) const;
  bool link_output_compute(Dep<OutputCompute>& output);
  void unlink_output_compute();
  const Dep<OutputCompute>& get_output_compute() const;
  bool is_connected() const;

  // Serialization.
  virtual void save(SimpleSaver& saver) const;
  virtual void load(SimpleLoader& loader);

 protected:
  // Our state.
  virtual bool update_state();

 private:

  // Our dynamic deps.
  Dep<OutputCompute> _upstream;

  // Our data.
  QJsonValue _unconnected_value;

  // Whether we are exposed in the node graph as a plug on a node.
  bool _exposed;
};
{% endhighlight %}

Input base class implementation.

{% highlight cpp %}
InputCompute::InputCompute(Entity* entity)
    : Compute(entity, kDID()),
      _upstream(this),
      _exposed(true) {
	get_dep_loader()->register_dynamic_dep(_upstream);
}

InputCompute::~InputCompute() {
}

bool InputCompute::update_state() {
  internal();
  Compute::update_state();

  // This will hold our final output value.
  QJsonValue output;

  // Start with our unconnected value.
  output = _unconnected_value;

  // Merge in information from the upstream output.
  if (_upstream) {
    QJsonValue out = _upstream->get_main_output();
    output = JSONUtils::deep_merge(output, out);
  }

  // Cache the result in our outputs.
  set_main_output(output);
  return true;
}

void InputCompute::set_unconnected_value(const QJsonValue& value) {
  external();
  _unconnected_value = value;
}

const QJsonValue& InputCompute::get_unconnected_value() const {
  external();
  return _unconnected_value;
}

void InputCompute::set_exposed(bool exposed) {
  external();
  _exposed = exposed;
}

bool InputCompute::is_exposed() const {
  external();
  return _exposed;
}

bool InputCompute::can_link_output_compute(const Dep<OutputCompute>& output) const {
  external();
  assert(output);
  if (dep_creates_cycle(output)) {
    return false;
  }
  return true;
}

bool InputCompute::link_output_compute(Dep<OutputCompute>& output) {
  external();
  assert(output);

  // Make sure to be disconnected before connecting.
  if (!_upstream && output) {
    unlink_output_compute();
  }
  // Now connect.
  _upstream = output;
  if (_upstream) {
    return true;
  }
  // Linking was unsuccessful.
  return false;
}

const Dep<OutputCompute>& InputCompute::get_output_compute() const {
  external();
  return _upstream;
}

bool InputCompute::is_connected() const {
  if (_upstream) {
    return true;
  }
  return false;
}

void InputCompute::unlink_output_compute() {
  external();
  if (_upstream) {
    _upstream.reset();
  }
}

void InputCompute::save(SimpleSaver& saver) const {
  external();
  Compute::save(saver);

  // Serialize the exposed value.
  saver.save(_exposed);

  // Serialize the value.
  QByteArray data = JSONUtils::serialize_json_value(_unconnected_value);
  int num_bytes = data.size();
  saver.save(num_bytes);
  saver.save_raw(data.data(), num_bytes);
}

void InputCompute::load(SimpleLoader& loader) {
  external();
  Compute::load(loader);

  // Load the exposed value.
  loader.load(_exposed);

  // Load the num bytes of the param value.
  int num_bytes;
  loader.load(num_bytes);

  // Load the param value.
  QByteArray data;
  data.resize(num_bytes);
  loader.load_raw(data.data(), num_bytes);
  _unconnected_value = JSONUtils::deserialize_json_value(data);
}
{% endhighlight %}

