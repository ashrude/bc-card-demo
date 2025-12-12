<script lang="ts">
	import { convert_js_image_to_luma, convert_canvas_to_luma, decode_barcode_with_hints, decode_multi, DecodeHintDictionary, DecodeHintTypes, BarcodeFormat } from "rxing-wasm";
	
	//const aamva = require('aamva') as any;

	let canvas: HTMLCanvasElement;
	let decoded = $state("");
	let parsed_data = $state("");

	function handleFiles(e: any) {

		console.log(e);
		const img = new Image;
		const ctx = canvas.getContext('2d')
		img.src = URL.createObjectURL(e.target.files[0])
		img.onload = function() {
			canvas.width = img.width;
			canvas.height = img.height;
			
			ctx?.drawImage(img, 0, 0, img.width, img.height);
		}
	}

	function scan() {
		let luma_data;
		try {
			luma_data = convert_canvas_to_luma(canvas);
		} catch {
			alert("Failure converting image")
			return;
		}
		let result;
		try {
			let hints = new DecodeHintDictionary();

			result = decode_barcode_with_hints(luma_data, canvas.width, canvas.height, hints, true);
		} catch {
			alert("Failure decoding data")
			return;
		}

		console.log(result.text())
		decoded = result.text();
		parsed_data = JSON.stringify(decode(decoded), null, 4);
	}
	class Parser {
		text: string;
		cursor: number;
		
		constructor(text: string) {
			this.text = text;
			this.cursor = 0;
		}

		consume_fixed(length: number): string {
			let res = this.text.substring(this.cursor, this.cursor + length);
			this.cursor += length;
			return res;
		}
		consume_fixed_blank(length: number): string {
			let res = this.text.substring(this.cursor, this.cursor + length);
			this.cursor += length + 1;
			return res;
		}
		consume_var(limit: number, diliminator: string): string {
			let range = this.text.substring(this.cursor, this.cursor + limit);
			let end;
			let next;
			if (range.indexOf(diliminator) === -1) {
				end = this.cursor + limit;
				next = end;
			} else {
				end = this.cursor + range.indexOf(diliminator);
				next = end + 1;
			}
			//let end = this.cursor + (range.indexOf(diliminator) === -1 ? limit : range.indexOf(diliminator));
			let res = this.text.substring(this.cursor, end);
			this.cursor = next;
			
			return res;
		}
		consume_blanks(limit: number) {
			let range = this.text.substring(this.cursor, this.cursor + limit);
			let i = 0;
			while ( i < limit ) {
				if (range.at(i) !== " ") {
					this.cursor = this.cursor + i;
					return;
				}
				
				i ++;
			}
		}
	}

	type data = {
			t1: string,
			province: string,
			city: string,
			name: string,
			addr: string,

			t2: string,
			id: string,
			card_expiry: string,
			birthdate: string,
			t2_end: string,

			t3: string,
			version: string,
			security_version: string,
			postal_code: string,
			_: void,
			
			sex_height: string,
			_blank: void,
			detail_clump: string,

			_lastblank: void,
			ecc: string,
			t3_end: string,
	}

	type parsed = {
		province: string,
		city: string,
		name: string,
		addr: string
		card_expiry: string,
		birth_date: string,
		postal_code: string,

		sex: string,
		height: number | null,

		clump: string,
		phn: string | null,
		weight: number | null,
		eye_colour: string | null,
		hair_colour: string | null,

	}

	function decode_basic(text: string): data {
		let thing = new Parser(text);
		let data: data = {
			t1: thing.consume_fixed(1),
			province: thing.consume_fixed(2),
			city: thing.consume_var(13, "^"),
			name: thing.consume_var(35, "^"),
			addr: thing.consume_var(42, "?"),

			t2: thing.consume_fixed(1),
			id: thing.consume_var(20, "="),
			card_expiry: thing.consume_fixed(4),
			birthdate: thing.consume_fixed(8),
			t2_end: thing.consume_var(2, "?"),

			t3: thing.consume_fixed(2),
			version: thing.consume_fixed(1),
			security_version: thing.consume_fixed(1),
			postal_code: thing.consume_fixed_blank(6),
			_: thing.consume_blanks(50),
			
			sex_height: thing.consume_var(4, " "),
			_blank: thing.consume_blanks(50),
			detail_clump: thing.consume_var(20, " "),

			_lastblank: thing.consume_blanks(50),
			ecc: thing.consume_fixed(11),
			t3_end: thing.consume_fixed(1)


		}
		
		return data;
	}

	function decode(text: string): parsed {
		let basic = decode_basic(text);

		let sex = basic.sex_height[0];
		let height = basic.sex_height.substring(1,8) === "" ? null : Number(basic.sex_height.substring(1,8));
		let phn: string | null = null;
		let details: string | null = null;

		if (basic.detail_clump.length < 10) { // Details only
			details = basic.detail_clump
		} else if (basic.detail_clump.length > 10) { // Details and phn
			phn = basic.detail_clump.substring(basic.detail_clump.length - 10);
			details = basic.detail_clump.substring(0, basic.detail_clump.length - 10)
			
		} else if (basic.detail_clump.length === 10) { // PHN Only
			phn = basic.detail_clump
		}

		return {
			province: basic.province,
			city: basic.city,
			name: basic.name,
			addr: basic.addr,
			card_expiry: basic.card_expiry,
			birth_date: basic.birthdate,
			postal_code: basic.postal_code,
			sex: sex,
			height: height,

			clump: basic.detail_clump,
			phn: phn,
			weight: Number(details?.substring(0, details.length - 6)),
			eye_colour: details?.substring(details.length - 6, details.length - 3) ? details?.substring(details.length - 6, details.length - 3) : null,
			hair_colour: details?.substring(details.length - 3, details.length) ? details?.substring(details.length - 3, details.length) : null,
		}
	}
</script>

<h1>BC Card Decoder demo</h1>
<p>This is demo, all data is processed on-device. It uses <a href="https://github.com/rxing-core/rxing-wasm/tree/main">rxing-wasm</a> and a rudimentary aamva parser, specs found <a href="https://www2.gov.bc.ca/assets/gov/health/practitioner-pro/medical-services-plan/ch4.pdf">here.</a></p>
<p>by Ashley Rudelsheim at <a href="https://itsphere.ca">ITSphere</a></p>

<div>
	<canvas bind:this={canvas}></canvas>
	<div>
		<input type="file" name="Image" id="Image" onchange={handleFiles}>
		<button onclick={scan}>Scan</button>
	</div>
</div>

<div>
	<span>Raw Data: </span>
	<p class="data">{decoded}</p>
</div>
<div>
	<span>Parsed Data: </span>
	<p class="data">{parsed_data}</p>
</div>

<style>
	canvas {
		width: 25rem;
	}
	.data {
		white-space: pre;
		font-family: monospace;
		font-size: larger;
	}
	
</style>