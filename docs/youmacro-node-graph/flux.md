---
layout: default
title: Flux
parent: YouMacro Node Graph
nav_order: 4
---

# Flux
{: .no_toc }

The Flux base class gathers all the inputs or all the outputs for a given node. Parameters on a node are considered to be inputs to a node.

Flux base class header.

{% highlight cpp %}
struct InputTraits {
  typedef InputCompute IOCompute;
  static const EntityDID IOComputeEntityDID = EntityDID::kInputEntity;
  static const ComponentIID FluxIID = ComponentIID::kIInputs;
  static const ComponentDID FluxDID = ComponentDID::kInputs;
  static const char* folder_name;
};

struct OutputTraits {
  typedef OutputCompute IOCompute;
  static const EntityDID IOComputeEntityDID = EntityDID::kOutputEntity;
  static const ComponentIID FluxIID = ComponentIID::kIOutputs;
  static const ComponentDID FluxDID = ComponentDID::kOutputs;
  static const char* folder_name;
};

//--------------------------------------------------------------------------
// The Flux base class maintains a record of either the inputs or outputs of
// a nodal compute.
//--------------------------------------------------------------------------

template <class Traits>
class COMPUTES_EXPORT Flux: public Component {
 public:
  static ComponentIID kIID() {
      return Traits::FluxIID;
  }
  static ComponentDID kDID() {
    return Traits::FluxDID;
  }

  Flux(Entity* entity);
  virtual ~Flux();

  // Our relative positioning.
  virtual size_t get_exposed_index(const std::string& name) const;
  virtual size_t get_num_exposed() const;
  virtual size_t get_num_hidden() const;
  virtual size_t get_total() const;

  virtual const std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& get_hidden() const;
  virtual const std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& get_exposed() const;
  virtual const std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& get_all() const;

  virtual bool has(const std::string& name) const;


 protected:

  virtual const Dep<typename Traits::IOCompute>& get(const std::string& name) const;

  // Our state.
  virtual void update_wires();
  virtual void gather(std::unordered_map<std::string, size_t>& exposed_ordering,
                      std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& hidden,
                      std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& exposed,
                      std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& all);

  Dep<typename Traits::IOCompute> _null;
  Dep<BaseNodeGraphManipulator> _manipulator;

  // Alphabetical ordering of exposed inputs/outputs.
  std::unordered_map<std::string, size_t> _exposed_ordering;
  std::unordered_map<std::string, Dep<typename Traits::IOCompute> > _hidden;
  std::unordered_map<std::string, Dep<typename Traits::IOCompute> > _exposed;
  std::unordered_map<std::string, Dep<typename Traits::IOCompute> > _all;
};
{% endhighlight %}

Output base class implementation.

{% highlight cpp %}
const char* InputTraits::folder_name = kInputsFolderName;
const char* OutputTraits::folder_name = kOutputsFolderName;


template<class Traits>
Flux<Traits>::Flux(Entity* entity)
    : Component(entity, kIID(), kDID()),
      _manipulator(this),
      _null(this) {
  get_dep_loader()->register_fixed_dep(_manipulator, Path());
}

template<class Traits>
Flux<Traits>::~Flux() {
}

template<class Traits>
void Flux<Traits>::update_wires() {
  internal();

  std::unordered_map<std::string, size_t> next_exposed_ordering;
  std::unordered_map<std::string, Dep<typename Traits::IOCompute> > next_hidden;
  std::unordered_map<std::string, Dep<typename Traits::IOCompute> > next_exposed;
  std::unordered_map<std::string, Dep<typename Traits::IOCompute> > next_all;
  gather(next_exposed_ordering, next_hidden, next_exposed, next_all);

  // If we're already up to date, then return right away.
  if (next_exposed_ordering == _exposed_ordering &&
      next_hidden == _hidden &&
      next_exposed == _exposed &&
      next_all == _all) {
    return;
  }

  // Update our values.
  _exposed_ordering = next_exposed_ordering;
  _hidden = next_hidden;
  _exposed = next_exposed;
  _all = next_all;

  // Update the topologies.
  if (Traits::FluxDID == ComponentDID::kOutputs) {
    _manipulator->set_output_topology(our_entity(), _exposed_ordering);
  } else if (Traits::FluxDID == ComponentDID::kInputs) {
    _manipulator->set_input_topology(our_entity(), _exposed_ordering);
  }
}

template<class Traits>
void Flux<Traits>::gather(std::unordered_map<std::string, size_t>& exposed_ordering,
                          std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& hidden,
                          std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& exposed,
                          std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& all) {
  internal();

  exposed_ordering.clear();
  hidden.clear();
  exposed.clear();
  exposed_ordering.clear();

  std::vector<std::string> exposed_names;

  // Loop through all the inputs or outputs.
  Entity* space = has_entity(Path({".",Traits::folder_name}));
  if (space) {
    const Entity::NameToChildMap& children = space->get_children();
    for (auto iter: children) {
      Entity* child = iter.second;
      // If the child is not an input or output, then skip it.
      if (child->get_did() != Traits::IOComputeEntityDID) {
        continue;
      }
      // Grab a dep on the input or output.
      Dep<typename Traits::IOCompute> dep = get_dep<typename Traits::IOCompute>(child);
      if (dep) {
        if (dep->is_exposed()) {
          // If the input or output is exposed, record it in the exposed set.
          exposed.insert({child->get_name(),dep});
          exposed_names.push_back(child->get_name());
        } else {
          // If the input or output is not exposed, record it in the hidden set.
          hidden.insert({child->get_name(),dep});
        }
      }
    }
  }

  // Sort the exposed inputs/outputs alphabetically.
  std::sort(exposed_names.begin(), exposed_names.end());
  for (size_t i=0; i<exposed_names.size(); ++i) {
    exposed_ordering[exposed_names[i]] = i;
  }

  // Cache a merged map of the exposed and hidden.
  all = exposed;
  all.insert(hidden.begin(), hidden.end());
}

template<class Traits>
size_t Flux<Traits>::get_exposed_index(const std::string& name) const {
  external();
  if (_exposed_ordering.count(name)) {
    return _exposed_ordering.at(name);
  }
  return -1;
}

template<class Traits>
size_t Flux<Traits>::get_num_exposed() const {
  external();
  return _exposed.size();
}

template<class Traits>
size_t Flux<Traits>::get_num_hidden() const {
  external();
  return _hidden.size();
}

template<class Traits>
size_t Flux<Traits>::get_total() const {
  external();
  return _exposed.size() + _hidden.size();
}

template<class Traits>
const std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& Flux<Traits>::get_hidden() const {
  external();
  return _hidden;
}

template<class Traits>
const std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& Flux<Traits>::get_exposed() const {
  external();
  return _exposed;
}

template<class Traits>
const std::unordered_map<std::string, Dep<typename Traits::IOCompute> >& Flux<Traits>::get_all() const {
  external();
  return _all;
}

template<class Traits>
bool Flux<Traits>::has(const std::string& name) const {
  external();
  if (_all.count(name)) {
    return true;
  }
  return false;
}

template<class Traits>
const Dep<typename Traits::IOCompute>& Flux<Traits>::get(const std::string& name) const {
  external();
  if (_all.count(name)) {
    return _all.at(name);
  }
  return _null;
}
{% endhighlight %}

