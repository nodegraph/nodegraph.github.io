---
layout: default
title: While Node
parent: YouMacro Node Graph
nav_order: 9
---

# While Node
{: .no_toc }

An While Node is an example of a flow control node. A While Node is a group node, that continuously executes its internal subgraph until certain conditions are met. When the conditions are met the final output values are recorded in the While node's outputs. In this example the conditions are expressed using Javascript.

While node base class header.

{% highlight cpp %}
class COMPUTES_EXPORT WhileGroupNodeCompute: public GroupNodeCompute {
 public:
  COMPONENT_ID(Compute, WhileGroupNodeCompute);
  WhileGroupNodeCompute(Entity* entity, ComponentDID did = kDID());
  virtual ~WhileGroupNodeCompute();

 protected:
  virtual void reset_loop_context();

  // Our state.
  virtual bool update_state();
  virtual void init_dirty_state();

  size_t _infinite_counter;

  bool _do_next;
};
{% endhighlight %}

While node base class implementation.

{% highlight cpp %}
WhileGroupNodeCompute::WhileGroupNodeCompute(Entity* entity, ComponentDID did):
    GroupNodeCompute(entity, did),
    _infinite_counter(0),
    _do_next(false) {
}

WhileGroupNodeCompute::~WhileGroupNodeCompute() {
}

void WhileGroupNodeCompute::reset_loop_context() {
  // Reset our script loop context.
  Dep<ScriptLoopContext> c = get_dep<ScriptLoopContext>(Path({"."}));
  c->reset();

  // Reset the script loop context on any child group nodes.
  const Entity::NameToChildMap &children = our_entity()->get_children();
  for (auto &iter: children) {
    if (iter.second->get_did() == EntityDID::kWhileGroupNodeEntity) {
      Dep<WhileGroupNodeCompute> w = get_dep<WhileGroupNodeCompute>(iter.second);
      w->reset_loop_context();
    }
  }
}

void WhileGroupNodeCompute::init_dirty_state() {
  GroupNodeCompute::init_dirty_state();
  reset_loop_context();
  _do_next = true;
}

bool WhileGroupNodeCompute::update_state() {
  internal();
  Compute::update_state();

  // Get the value of the condition.
  QJsonObject in_obj = _inputs->get_main_input_object();
  QJsonValue condition_value = in_obj.value("value");
  bool condition = JSONUtils::convert_to_bool(condition_value);

  // If the "condition" input is false then we copy the value from the main input to the main output.
  // We set the value to zero for all other outputs.
  if (!condition) {
    Entity* outputs = get_entity(Path( { ".", kOutputsFolderName }));
    for (auto &iter : outputs->get_children()) {
      Entity* output_entity = iter.second;
      const std::string& output_name = output_entity->get_name();
      Entity* output_node = our_entity()->get_child(output_name);
      Dep<OutputNodeCompute> output_node_compute = get_dep<OutputNodeCompute>(output_node);
      if (output_name == kMainOutputName) {
        // We copy the value from in to out.
        set_main_output(_inputs->get_main_input_object());
      } else {
        // We set the value to zero for all other outputs.
        set_output(output_name, 0);
      }
    }
    return true;
  }

  while (true) {
    // Set the inputs dirty when starting an iteration of the loop,
    // so that we can perform the compute again.
    if (_do_next) {
      for (auto &iter: _inputs->get_all()) {
        // Name of input on the group.
        std::string input_name = iter.second->get_name();
        // Find the input compute inside the group.
        Entity* input_node = our_entity()->get_child(input_name);
        assert(input_node);
        Dep<Compute> input_compute = get_dep<Compute>(input_node);
        input_compute->dirty_state();
      }
      _do_next = false;
    }

    // Run the regular group compute.
    if (!GroupNodeCompute::update_state()) {
      return false;
    }

    // Check the main output value to see if the value at condition_path is false.
    QJsonObject out_obj = get_main_output().toObject();
    QJsonValue condition_value = out_obj.value("value");
    if (!condition_value.toBool()) {
      break;
    }

    // We return false once in a while so the user can stop infinite while loops.
    if ((_infinite_counter++ % 2) == 0) {
      _manipulator->continue_cleaning_to_ultimate_targets_on_idle();
      return false;
    }

    // If we've made it here we successfully go to the next element.
    // This logic is needed because some of the internal nodes may return false, in the
    // above GroupNodeCompute::update_state(). In this case we are not ready to move onto
    // the next element yet.
    _do_next = true;
  }
  _do_next = false;
  return true;
}
{% endhighlight %}

