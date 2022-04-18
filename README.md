## rust-portaudio Recording
* reference -> https://github.com/RustAudio/rust-portaudio/blob/master/examples/blocking.rs

```rust
extern crate portaudio;
use portaudio as pa;
use std::collections::VecDeque;

const SAMPLE_RATE: f64 = 16000.0;
const CHANNELS: i32 = 1;
const FRAMES: u32 = 16000;
const INTERLEAVED: bool = true;

fn main() {
    match run() {
        Ok(_) => {}
        e => {
            eprintln!("Example failed with the following: {:?}", e);
        }
    }
}

fn run() -> Result<(), pa::Error> {
    let pa = pa::PortAudio::new()?;

    let def_input = pa.default_input_device()?;
    let input_info = pa.device_info(def_input)?;
    let latency = input_info.default_low_input_latency;
    let input_params = 
        pa::StreamParameters::<f32>::new(
            def_input, 
            CHANNELS, 
            INTERLEAVED, 
            latency);
    let settings = pa::InputStreamSettings::new(input_params, SAMPLE_RATE, FRAMES);
    let mut stream = pa.open_blocking_stream(settings)?;
    
    let mut buffer: VecDeque<f32> = VecDeque::with_capacity(FRAMES as usize * CHANNELS as usize);

    stream.start()?;
    
    fn wait_for_stream<F>(f: F, name: &str) -> u32
    where
        F: Fn() -> Result<pa::StreamAvailable, pa::error::Error>,
    {
        loop {
            match f() {
                Ok(available) => match available {
                    pa::StreamAvailable::Frames(frames) => return frames as u32,
                    pa::StreamAvailable::InputOverflowed => println!("Input stream has overflowed"),
                    pa::StreamAvailable::OutputUnderflowed => {
                        println!("Output stream has underflowed")
                    }
                },
                Err(err) => panic!(
                    "An error occurred while waiting for the {} stream: {}",
                    name, err
                ),
            }
        }
    }

    loop {
        let in_frames = wait_for_stream(|| stream.read_available(), "Read");

        if in_frames > 0 {
            let input_samples = stream.read(in_frames)?;
            buffer.extend(input_samples.into_iter());
        }
    }
}
```