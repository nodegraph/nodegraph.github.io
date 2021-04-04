---
layout: default
title: Compute
parent: YouMacro Node Graph
nav_order: 5
---

# Compute
{: .no_toc }

The Compute base class is the base logic for perform a given node's computation. The Compute base class contains all the outputs (data values) for a give node. The inputs to the compute are retrieved from the outputs on other compute instances, or from parameters on the node. Parameters on a node are considered to be inputs to a node.

Compute base class header.

{% highlight cpp %}
// The main input and main output are known to our nodegraph system
// and various logic uses that knowledge.
// Other inputs and outputs are only used within that specific compute.
class COMPUTES_EXPORT Compute: public Component {
 public:

  static const char* kMainInputName;
  static const char* kMainOutputName;

  static const char* kValuePropertyName;

  COMPONENT_ID(Compute, InvalidComponent);
  Compute(Entity* entity, ComponentDID derived_id);
  virtual ~Compute();

  virtual void create_inputs_outputs(const EntityConfig& config);
  virtual void set_self_dirty(bool dirty);

  virtual const Dep<Inputs>& get_inputs() {return _inputs;}

  // Inputs.
  virtual QJsonObject get_editable_inputs() const;
  virtual void set_editable_inputs(const QJsonObject& inputs);
  virtual QJsonObject get_input_exposure() const;
  virtual void set_input_exposure(const QJsonObject& settings);

  // Access outputs.
  virtual const QJsonObject& get_outputs() const;
  virtual QJsonValue get_output(const std::string& name) const;
  virtual QJsonValue get_main_output() const;

  // Get our hints.
  virtual const QJsonObject& get_hints() const {return _hints;}

  bool eval_js_with_inputs(const QString& text, QJsonValue& result, QString& error) const;

  void on_error(const QString& error_message);


  virtual bool update_unlocked_group() {return true;}

 protected:
  // Our state.
  virtual void initialize_wires();
  virtual bool update_state();

  virtual bool clean_finalize();

  // Our outputs. These are called during cleaning, so they don't dirty the instance's state.
  virtual void set_outputs(const QJsonObject& outputs);
  virtual void set_output(const std::string& name, const QJsonValue& value);
  virtual void set_main_output(const QJsonValue& value);

  // Plugs.
  Entity* create_input(const std::string& name, const EntityConfig& config);
  Entity* create_output(const std::string& name, const EntityConfig& config);

  Entity* create_main_input(const EntityConfig& config);
  Entity* create_main_output(const EntityConfig& config);

  Entity* create_namespace(const std::string& name);
  Entity* get_inputs_space();
  Entity* get_outputs_space();
  Entity* get_links_space();

  // Used by derived classes.
  static void add_hint(QJsonObject& map, const std::string& name, GUITypes::HintKey hint_type, const QJsonValue& value);
  static void remove_hint(QJsonObject& node_hints, const std::string& name);
  static void add_main_input_hint(QJsonObject& map);

 protected:
  Dep<Inputs> _inputs;
  Dep<BaseNodeGraphManipulator> _manipulator;

  // Our outputs.
  QJsonObject _outputs;

  // Hints are generally used for all inputs on a node.
  // They are used to display customized guis for the input.
  // Note this maps the input name to hints.
  // The key is the string of the number representing the HintType.
  // The value is a QJsonValue holding an int representing an enum or another type.
  static const QJsonObject _hints;

};
{% endhighlight %}

Compute base class implementation.

{% highlight cpp %}
const char* Compute::kMainInputName = "in";
const char* Compute::kMainOutputName = "out";

const char* Compute::kValuePropertyName = "value";

struct InputComputeComparator {
  bool operator()(const Dep<InputCompute>& left, const Dep<InputCompute>& right) const {
    return left->get_name() < right->get_name();
  }
};

const QJsonObject Compute::_hints;

Compute::Compute(Entity* entity, ComponentDID derived_id)
    : Component(entity, kIID(), derived_id),
      _inputs(this),
      _manipulator(this) {
  // Note this only exists for node computes and not for plug computes.
  get_dep_loader()->register_fixed_dep(_inputs, Path({"."}));
  // We only grab the manipulator for non input/output computes, to avoid cycles.
}

Compute::~Compute() {
}

void Compute::create_inputs_outputs(const EntityConfig& config) {
  external();
  create_namespace(kInputsFolderName);
  create_namespace(kOutputsFolderName);
}

void Compute::set_self_dirty(bool dirty) {
  Component::set_self_dirty(dirty);
  if (_manipulator) {
      std::cerr << to_underlying(get_did()) << " Compute setting marker to: " << !is_state_dirty() << "\n";
      _manipulator->update_clean_marker(our_entity(), !is_state_dirty());
      if (dirty) {
        std::cerr << to_underlying(get_did()) << " Compute is bubbling dirtiness\n";
        _manipulator->bubble_group_dirtiness(our_entity());
      }
  }
}

void Compute::initialize_wires() {
  Component::initialize_wires();
  if ((get_did() != ComponentDID::kInputCompute) && (get_did() != ComponentDID::kOutputCompute)) {
    _manipulator = get_dep<BaseNodeGraphManipulator>(get_app_root());
  }
}

bool Compute::update_state() {
  internal();
  // Notify the gui side that a computation is now processing on the compute side.
  if (_manipulator) {
    _manipulator->set_processing_node(our_entity());
  }

  return true;
}

bool Compute::clean_finalize() {
  internal();
  Component::clean_finalize();
  // Notify the gui side that a computation is now processing on the compute side.
  //if (_manipulator) {
  //  _manipulator->clear_processing_node();
  //}
  return true;
}

QJsonObject Compute::get_editable_inputs() const {
  return _inputs->get_editable_inputs();
}

void Compute::set_editable_inputs(const QJsonObject& inputs) {
  _inputs->set_editable_inputs(inputs);
}

QJsonObject Compute::get_input_exposure() const {
  return _inputs->get_exposure();
}

void Compute::set_input_exposure(const QJsonObject& settings) {
  _inputs->set_exposure(settings);
}

const QJsonObject& Compute::get_outputs() const {
  external();
  return _outputs;
}

void Compute::set_outputs(const QJsonObject& outputs) {
  internal();
  _outputs = outputs;
}

QJsonValue Compute::get_output(const std::string& name) const{
  external();
  if (!_outputs.contains(name.c_str())) {
    return QJsonValue();
  }
  return _outputs.value(name.c_str());
}

QJsonValue Compute::get_main_output() const {
  return get_output(kMainOutputName);
}

void Compute::set_output(const std::string& name, const QJsonValue& value) {
  internal();
  _outputs.insert(name.c_str(), value);
}

void Compute::set_main_output(const QJsonValue& value) {
  internal();
  _outputs.insert(kMainOutputName, value);
}

Entity* Compute::create_input(const std::string& name, const EntityConfig& config) {
  external();

  // Get the inputs namespace.
  Dep<BaseFactory> factory = get_dep<BaseFactory>(Path());
  Entity* inputs_space = get_inputs_space();

  // Make sure the name doesn't exist already.
  assert(!inputs_space->get_child(name));

  // Create the input.
  InputEntity* input = static_cast<InputEntity*>(factory->instance_entity(inputs_space, EntityDID::kInputEntity, name));
  input->create_internals(config);
  return input;
}

Entity* Compute::create_output(const std::string& name, const EntityConfig& config) {
  external();

  // Get the outputs namespace.
  Dep<BaseFactory> factory = get_dep<BaseFactory>(Path());
  Entity* outputs_space = get_outputs_space();

  // Make sure the name doesn't exist already.
  assert(!outputs_space->get_child(name));

  // Create the output.
  OutputEntity* output = static_cast<OutputEntity*>(factory->instance_entity(outputs_space, EntityDID::kOutputEntity, name));
  output->create_internals(config);
  return output;
}

Entity* Compute::create_main_input(const EntityConfig& config) {
  EntityConfig c = config;
  c.unconnected_value = QJsonObject();
  return create_input(kMainInputName, c);
}

Entity* Compute::create_main_output(const EntityConfig& config) {
  EntityConfig c = config;
  c.unconnected_value = QJsonObject();
  return create_output(kMainOutputName, c);
}

Entity* Compute::create_namespace(const std::string& name) {
  external();
  Dep<BaseFactory> factory = get_dep<BaseFactory>(Path());
  Entity* space = static_cast<BaseNamespaceEntity*>(factory->instance_entity(our_entity(), EntityDID::kBaseNamespaceEntity, name));
  space->create_internals();
  return space;
}

Entity* Compute::get_inputs_space() {
  external();
  return our_entity()->get_child(kInputsFolderName);
}

Entity* Compute::get_outputs_space() {
  external();
  return our_entity()->get_child(kOutputsFolderName);
}

Entity* Compute::get_links_space() {
  external();
  return our_entity()->get_child(kLinksFolderName);
}

void Compute::add_hint(QJsonObject& node_hints, const std::string& name, GUITypes::HintKey hint_type, const QJsonValue& value) {
  QJsonObject param_hints = node_hints.value(name.c_str()).toObject();
  param_hints.insert(QString::number(to_underlying(hint_type)), value);
  node_hints.insert(name.c_str(),  param_hints);
}

void Compute::remove_hint(QJsonObject& node_hints, const std::string& name) {
  node_hints.remove(name.c_str());
}

void Compute::add_main_input_hint(QJsonObject& map) {
  add_hint(map, kMainInputName, GUITypes::HintKey::DescriptionHint, "The main input object that the node operates on.");
}

void Compute::on_error(const QString& error_message) {
  _manipulator->set_error_node(error_message);
  _manipulator->clear_ultimate_targets();
}

bool Compute::eval_js_with_inputs(const QString& text, QJsonValue& result, QString& error) const {
  internal();
  QJSEngine engine;

  // Add the input values into the context.
  for (auto &iter: _inputs->get_all()) {
    const Dep<InputCompute>& input = iter.second;
    QJsonValue value = input->get_main_output();
    const std::string& input_name = input->get_name();
    QJSValue jsvalue = engine.toScriptValue(value);
    engine.globalObject().setProperty(QString::fromStdString(input_name), jsvalue);
  }

  return JSONUtils::eval_js_in_context(engine, text, result,error);
}
{% endhighlight %}

