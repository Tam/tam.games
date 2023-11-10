+++
title = "My first blog post"
date = 2023-11-10
+++

some blog content

```rust
use std::f32::consts::{PI, TAU};

#[derive(Debug)]
pub struct Behaviour {
	gradient: Vec<[f32;2]>,
	ad_cache: Box<[f32]>,
	resolution: usize,
	sigma: f32,
	blending: f32,
	cached_heading: Option<(f32, f32)>,
}

impl Behaviour {

	/// Create a new Behaviour struct
	///
	/// * `resolution` - The number of points around a circle to store interest & danger values.
	///     Use a higher resolution for a more accurate output, but at the cost of more stored and
	///     processed data. The lowest recommended value is 8 to cover all cardinal and intermediate
	///     directions.
	/// * `sigma` - The gaussian distribution modifier, try 0.5?
	/// * `sample_neighbours` - The number of neighbours to sample when calculating the best
	///     direction
	/// * `blending` - The amount to blend the previous frame with the next
	pub fn new(resolution: usize, sigma: f32, blending: f32) -> Self {
		// Cache the non-changing values used in the angular distance calculation
		let mut ad_cache = Vec::with_capacity(resolution);
		for i in 0..resolution {
			ad_cache.push((i as f32) / (resolution as f32) * TAU);
		}

		Self {
			gradient: vec![[0., 0.]; resolution],
			ad_cache: ad_cache.into_boxed_slice(),
			resolution,
			sigma,
			blending,
			cached_heading: None,
		}
	}

	/// Get the gradient values scaled from 0-1
	///
	/// Returns a vector of arrays, each containing [radians, interest, danger]
	pub fn get_gradient (&self) -> Vec<[f32; 3]> {
		self.gradient
			.clone()
			.iter()
			.enumerate()
			.map(|(i, [interest, danger])| [
				i as f32 / self.resolution as f32 * TAU,
				*interest,
				*danger,
			])
			.collect()
	}

	/// Add interest
	///
	/// * `radians` - the angle of interest
	/// * `value` - the value of interest, must be between 0 and 1
	pub fn add_interest (&mut self, radians: f32, value: f32) {
		self.set_value(0, radians, value);
	}

	/// Add danger
	///
	/// * `radians` - the angle of danger
	/// * `value` - the value of danger, must be between 0 and 1
	pub fn add_danger (&mut self, radians: f32, value: f32) {
		self.set_value(1, radians, value);
	}

	/// Set the value for a given type, automatically smoothing it out to it's neighbours
	///
	/// * `type_index` - 0 = interest, 1 = danger
	/// * `radians` - target angle
	/// * `value` - target value
	fn set_value (&mut self, type_index: usize, radians: f32, value: f32) {
		let value = value.clamp(0., 1.);

		for i in 0..self.resolution {
			let angular_distance = (self.ad_cache[i] - radians).abs();
			let shortest_distance = angular_distance.min(2. * PI - angular_distance);

			let weight = (-0.5 * (shortest_distance / self.sigma).powf(2.)).exp();
			let v = value * weight;

			self.gradient[i][type_index] = (self.gradient[i][type_index]).max(v);
		}
	}

	/// Computes the best heading and clears the gradient
	///
	/// Returns a tuple `(x, y)` emulating a unit vector, scaled to the intensity of the heading
	pub fn get_heading(&mut self) -> Option<(f32, f32)> {
		if self.cached_heading.is_some() {
			return self.cached_heading;
		}

		let merged_gradient = self.gradient.iter()
			.map(|p| (p[0] - p[1]).clamp(0., 1.))
			.collect::<Vec<_>>();

		// Convert radians to Cartesian coordinates
		let x_coords: Vec<f32> = self.ad_cache.iter().map(|&angle| f32::cos(angle)).collect();
		let y_coords: Vec<f32> = self.ad_cache.iter().map(|&angle| f32::sin(angle)).collect();

		// Apply weights
		let weighted_x: Vec<f32> = x_coords.iter().zip(merged_gradient.iter()).map(|(&x, &w)| x * w).collect();
		let weighted_y: Vec<f32> = y_coords.iter().zip(merged_gradient.iter()).map(|(&y, &w)| y * w).collect();

		// Sum the weighted coordinates
		let sum_weighted_x: f32 = weighted_x.iter().sum();
		let sum_weighted_y: f32 = weighted_y.iter().sum();

		self.cached_heading = Some((
			sum_weighted_x,
			sum_weighted_y,
		));

		self.cached_heading
	}

	/// Clear the gradient
	///
	/// This should be called at the beginning or end of each frame.
	/// The gradient will be blended with the previous frame using the `blending` factor.
	pub fn clear (&mut self) {
		for v in &mut self.gradient {
			v[0] *= self.blending;
			v[1] *= self.blending;
		}

		self.cached_heading = None;
	}

}
```
