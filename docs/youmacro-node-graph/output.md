---
layout: default
title: Output
parent: YouMacro Node Graph
nav_order: 3
---

# Output
{: .no_toc }

The Output base class allows quick access to output values from a given node's computation. Often nodes have many inputs and many outputs. Parameters on a node are considered to be inputs to a node.

Output base class header.

{% highlight cpp %}
class COMPUTES_EXPORT OutputCompute: public Compute {
 public:
  COMPONENT_ID(Compute, OutputCompute);
  OutputCompute(Entity* entity);
  virtual ~OutputCompute();

  // We override to stop creating the inputs and outputs namespace.
  virtual void create_inputs_outputs(const EntityConfig& config = EntityConfig()) {external();}

  // Exposure.
  void set_exposed(bool exposed);
  bool is_exposed() const;

  // Serialization.
  virtual void save(SimpleSaver& saver) const;
  virtual void load(SimpleLoader& loader);

 protected:

  // Our state.
  virtual bool update_state();

 private:
  // Our fixed deps.
  Dep<Compute> _node_compute;

  // Exposure.
  bool _exposed;

  friend class InputCompute;
  friend class OutputShape;
};
{% endhighlight %}

Output base class implementation.

{% highlight cpp %}
OutputCompute::OutputCompute(Entity* entity)
    : Compute(entity, kDID()),
      _node_compute(this),
      _exposed(true) {
  get_dep_loader()->register_fixed_dep(_node_compute, Path({"..",".."}));
}

OutputCompute::~OutputCompute() {
}

bool OutputCompute::update_state() {
  internal();
  Compute::update_state();

  // Using our name we query our nodes compute results.
  const std::string& our_name = our_entity()->get_name();
  set_main_output(_node_compute->get_output(our_name));
  return true;
}

void OutputCompute::set_exposed(bool exposed) {
  external();
  _exposed = exposed;
}

bool OutputCompute::is_exposed() const {
  external();
  return _exposed;
}

void OutputCompute::save(SimpleSaver& saver) const {
  external();
  Compute::save(saver);

  // Serialize the exposed value.
  saver.save(_exposed);
}

void OutputCompute::load(SimpleLoader& loader) {
  external();
  Compute::load(loader);

  // Load the exposed value.
  loader.load(_exposed);
}
{% endhighlight %}

