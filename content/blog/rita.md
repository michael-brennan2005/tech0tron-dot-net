+++
title = "Rita v0.1 prototype"
date = 2024-08-14
+++

## [Video demo](https://youtu.be/VqbNpBe7RlE)

Rita is an audio graph editor I've been working on this week, written with Rust on the backend, and React, Typescript, and TailwindCSS on the frontend. The idea for Rita is that you'll be able to create graphs to process audio signals and create sounds. What differentiates Rita from other DSP or sound synthesis tools is that Rita aims to be a _visual_ graph interface, in the same vein as Blender's material graphs or Unreal Engine's visual programming graphs. With a visual interface, anyone - musicians, music producers, people just getting interested in DSP - can create effects without having to become a programmer. As of now, Rita is in a very basic state, but there's enough at this point to talk about the architecture I've setup and the tech stack I'm using.

## Frontend

I've never been much of a frontend person, and so my original plan was to use something like egui or imgui which would handle most of the UI layout and styling work. But imgui's Rust bindings are a bit subpar, and [the only graph editor add-on for egui I could find is archived.](https://github.com/setzer22/egui_node_graph) It became clear quickly I was going to need to use something from the much-larger web ecosystem. Thankfully, Tauri is a framework made for desktop apps using Rust on the backend and HTML+CSS+Typescript on the frontend. Rita is now using [ReactFlow](https://reactflow.dev/), which has been really easy to use: the library handles all node and edge state, and makes it simple to add connectors to nodes (using the `Handle` element). It also serializes itself to JSON which can be easily sent to the Rust backend. The only tricky part of ReactFlow has been having to disable some of its built-in styling for my own.

Speaking of styling, I'm using TaliwindCSS for general styling and [shadcn/ui](https://ui.shadcn.com/) for most components. shadcn/ui is something I haven't used before this project, but it's been great for quickly building out the UI. What's unique about shadcn/ui is that it isn't built as a traditional library but more so meant to be copy/pasted into your codebase. This makes it simple to make changes to its components to your liking: I was able to modify the `Card` component, for example, to support the colored headers for the graph nodes.

Graph Processing
----------------

Getting the JSON data for the graph is just a matter of calling `JSON.stringify(rfInstance.toObject())`, where `rfInstance` is a state value holding the ReactFlowInstance. When the user hits compile, the frontend invokes a Tauri command, sending this JSON to the backend. From here, processing the graph is done in 3 parts: parsing the JSON into more usable data, processing the graph one node at a time, and then sending the final buffer of audio samples to the playback thread.

JSON parsing is done using serde, which involves defining Rust structs that conform to how ReactFlow's JSON is laid out. [These types are found in graph\_json.rs.](https://github.com/michael-brennan2005/rita-graph/blob/main/src-tauri/src/graph_json.rs) We then use this JSON data to create our `AudioGraph` struct:

```rust
pub struct AudioGraph {
  graph: petgraph::Graph<AudioGraphNode, AudioGraphEdge>
} 

enum AudioGraphNode {
    Input {
        file_path: String
    },
    WaveGen {
        wave_type: WaveType,
        frequency: f32,
        amplitude: f32,
        seconds: f32
    },
    BinOp {
        operation: BinOp,
        on_short_a: ShortBehavior,
        on_short_b: ShortBehavior
    },
    Output {
        final_buffer: Vec<f32>
    }
}

struct AudioGraphEdge {
  from_idx: usize,
  to_idx: usize
}

#[derive(Clone)]
enum EdgeData {
  SoundBuffer {
      buf: Vec<f32>
  }
}
```

Above is the definiton for the `AudioGraph` struct, with the definitions of `AudioGraphNode, AudioGraphEdge,` and `EdgeData` for the full picture. Right now, `AudioGraph` only holds a `petgraph::Graph` field, which stores all the nodes and edges. `AudioGraphNode` is (as you can probably guess) the node type of the graph, and stores all the parameters a node may need (minus any inputs that come from other nodes) for processing.

At this point you may be wondering where graph inputs and outputs are stored, and why `AudioGraphEdge` is, somewhat unintuitively, storing two `usize`s and nothing more. A key design choice I made was that `AudioGraphNode`s should not store their inputs or outputs; the vectors containing inputs and outputs are instead passed in when it comes time for a node to be processed. This is what `AudioGraphNode::process`'s signature is:

```rust
pub fn process(&mut self, 
    window: &tauri::Window, 
    my_idx: NodeIndex, 
    inputs: &mut HashMap<NodeIndex<u32>, Vec<(NodeIndex<u32>, usize, usize)>>, 
    outputs: &mut HashMap<NodeIndex<u32>, Vec<Option<EdgeData>>>,
    output_spec: F32FormatSpec
)
```   

To get the simple stuff out of the way, `window` is passed in to send status messages to the frontend using Tauri events; `my_idx` tells the node what index it should be using for `inputs` and `outputs`, and `output_spec` has info telling the node the sample rate and number of channels its final sound buffer should have.

`inputs` has an entry for each node (hence the `NodeIndex` as a key). The tuples are in the form of `(output_node_index, from_idx, to_idx)`: the node's (`to_idx`)th input is the (`output_node_index`)'s (`from_idx`)th output. This is what the `AudioGraphEdge` is storing - in `AudioGraph::process`, we use this edge info to create the `inputs` hashmap that eventually gets passed to all these nodes. Similar to `inputs`, `outputs` has an entry for each node; and the ith `Option<EdgeData>` is that node's ith output. Is this probably over-complicated and could be simplified? Yeah; but I'm probably going to be rewriting how inputs and outputs are handled when I switch to a streaming model (see below), and if I do stick with this way of storing them, it has the benefit that we can very easily support multiple inputs, multiple outputs, and (pretty sure) one output being used in more than one input.

Here's what `AudioGraph::process` - the 'main' method for graph processing - looks like. We topologically sort the graph (so nodes will always have their inputs ready before being processed), create the `inputs` and `outputs` maps, and then process the nodes one by one.

```rust
pub fn process(&mut self, window: &tauri::Window, output_spec: F32FormatSpec) -> Result<Vec<f32>, ()> {
  send_status(window, "Beginning graph processing.");

  // get sorted order
  let sorted = match petgraph::algo::toposort(&self.graph, None) {
      Ok(order) => order, 
      Err(_) => {
          send_status(window, "Error sorting graph - cycle exists");
          return Err(());
      },
  };
  let _ = window.emit("set_total_nodes", sorted.len());
  
  // create inputs map
  let mut inputs: HashMap<NodeIndex<u32>, Vec<(NodeIndex<u32>, usize, usize)>> = HashMap::new();
  for idx in &sorted {
      let mut vec: Vec<(NodeIndex<u32>, usize, usize)> = Vec::new();

      for edge in self.graph.edges_directed(*idx, Incoming) {
          vec.push((edge.source(), edge.weight().from_idx, edge.weight().to_idx));
      }

      inputs.insert(*idx, vec);
  }
  
  // create outputs map
  let mut outputs: HashMap<NodeIndex<u32>, Vec<Option<EdgeData>>> = HashMap::new();
  for idx in &sorted {
      let output_num = self.graph[*idx].num_outputs();

      let vec: Vec<Option<EdgeData>> = vec![None; output_num];
      
      outputs.insert(*idx, vec);
  }

  // Processing (the fun step)
  let _ = window.emit("set_completed_nodes", 0);
  for (i, idx) in sorted.iter().enumerate() {
      self.graph[*idx].process(window, *idx, &mut inputs, &mut outputs, output_spec);
      let _ = window.emit("set_completed_nodes", i + 1);
  }
  
  // Find output and return the final buffer
  for idx in &sorted {
      match &mut self.graph[*idx] {
          AudioGraphNode::Output { final_buffer } => {
              return Ok(std::mem::replace(final_buffer, Vec::new()));
          },
          _ => continue
      }
  }
  
  Err(())
}
```

[This code is all available in graph.rs.](https://github.com/michael-brennan2005/rita-graph/blob/main/src-tauri/src/graph.rs) Right now the audio graph is what I would call an "ahead-of-time" processor: its final output is the full sound buffer, start to end. For short sound buffers, this is fine, and the graph will finish processing in less than a second. When a graph starts using `wav` files however, compilation becomes very slow and the final sound buffer takes up a lot of memory. I've been using a `wav` file of Marvin Gaye's "What's Going On" during testing (great song! give it a listen) and compilation takes about 10 seconds and over 100 MB of memory; and that's just one song! I'm planning to rewrite the AudioGraph to use a just-in-time/streaming model, where the sound buffer is processed a chunk at a time instead of in one whole go.

## Playing audio

The functionality for audio playback is split into two structs: [`Player`](https://github.com/michael-brennan2005/rita-graph/blob/main/src-tauri/src/playback/player.rs) and [`Process`](https://github.com/michael-brennan2005/rita-graph/blob/main/src-tauri/src/playback/process.rs). They're both initialized in `Player::new`:

```rust
pub fn new() -> Player {
  // ... boilerplate cpal code ...
  let (app_to_player_send, app_to_player_recv) = RingBuffer::::new(256);
  let mut process = Process::new(app_to_player_recv, player_to_app_send);

  let stream = device
      .build_output_stream(
          &config,
          move |data: &mut [f32], _: &cpal::OutputCallbackInfo| {
              process.process(data);
          },
          move |err| {
              panic!("Audio ouput stream failure: {}", err);
          },
          None,
      )
      .expect("Stream building failed");

  stream.play().expect("Stream playing failed");

  Self {
      stream,
      format,
      app_to_player_send,
      player_to_app_recv,
  }
}
```

Audio playback is a realtime process: something like reading a file or a memory allocation taking longer than usual can cause very noticeable glitches or skips in the audio, which is why most playback libraries (such as [cpal, the library I'm using](https://github.com/RustAudio/cpal)) run the actual output-buffer filling code in a seperate thread. Thus, the returned `Player` struct is kept on the app thread, and can be used to pass messages to the `Process` struct that exists on the stream thread. The message types are below (I omitted the messages that are unused as of now), and are passed between the two structs with a [ringbuffer](https://docs.rs/rtrb/latest/rtrb/).

```rust
pub enum AppToPlayerMessage {
  Play,
  Pause,
  UseBuffer(Vec)
}
```

Using the `Player`, the app can tell the process to pause or play the current sound buffer, or swap it out for a new buffer (such as one returned from `AudioGraph::process`) to start playing from. `Process:process` (which gets called when the audio device needs new sound samples) will check for any incoming messages, and then just fill the output buffer with the nessecary samples:

```rust
if let Some(current_buffer) = &self.current_buffer {
    if self.current_buffer_idx + data.len() >= current_buffer.len() {
        let count_at_end = current_buffer.len() - self.current_buffer_idx;
        let count_at_start = data.len() - count_at_end;
        unsafe {
            std::ptr::copy(current_buffer.as_ptr().add(self.current_buffer_idx), data.as_mut_ptr(), count_at_end);
            std::ptr::copy(current_buffer.as_ptr(), data.as_mut_ptr().add(count_at_end), count_at_start);                    
        }
        self.current_buffer_idx = count_at_start;
    } else {
        unsafe {
            std::ptr::copy(current_buffer.as_ptr().add(self.current_buffer_idx), data.as_mut_ptr(), data.len());
        }
        self.current_buffer_idx += data.len();
    }
}
```

I did a quick benchmark to compare using iteration to fill the `data` buffer vs. `ptr::copy`, and I found `ptr::copy` faster enough to warrant using `unsafe` for it. The `Process` struct has a `current_buffer` field which is what it's reading samples from, and a `current_buffer_idx` to keep track of where it should start reading.

## Going forward
As I said a bit earlier, the main change I want to immediately make is switching to a just-in-time model to reduce loading times and memory usage. I'm heading back to college in a few weeks and likely won't have much time to work on this; my hope is that with this change and some upgrades to the UI, Rita would be in a nice place to gradually add new nodes or just come back to it when I have more free time.

[The full code for Rita is available on GitHub.](https://github.com/michael-brennan2005/rita-graph) If you have any feedback, suggestions, questions, feel free to reach out!