<script>
	//@ts-nocheck
	import Plotly from "plotly.js-dist-min";
	import { Plot as PlotIcon } from "@gradio/icons";
	import { Vega } from "svelte-vega";

	import { afterUpdate, onDestroy } from "svelte";

	export let value;
	export let target;
	let spec = null;

	$: if(value && value['type'] == "altair") {
		spec = JSON.parse(value['plot'])
	}


	// Plotly
	let plotDiv;	
	let plotlyGlobalStyle;

	const main_src = "https://cdn.bokeh.org/bokeh/release/bokeh-2.4.2.min.js";

	const plugins_src = [
		"https://cdn.pydata.org/bokeh/release/bokeh-widgets-2.4.2.min.js",
		"https://cdn.pydata.org/bokeh/release/bokeh-tables-2.4.2.min.js",
		"https://cdn.pydata.org/bokeh/release/bokeh-gl-2.4.2.min.js",
		"https://cdn.pydata.org/bokeh/release/bokeh-api-2.4.2.min.js"
	];

	function load_plugins() {
		return plugins_src.map((src, i) => {
			const script = document.createElement("script");
			script.onload = () => initializeBokeh(i + 1);
			script.src = src;
			document.head.appendChild(script);

			return script;
		});
	}

	function load_bokeh() {
		const script = document.createElement("script");
		script.onload = handleBokehLoaded;
		script.src = main_src;
		document.head.appendChild(script);

		return script;
	}

	function load_plotly_css() {
		if (!plotlyGlobalStyle) {
				plotlyGlobalStyle = document.getElementById("plotly.js-style-global");
				const plotlyStyleClone = plotlyGlobalStyle.cloneNode();
				target.appendChild(plotlyStyleClone);
				for (const rule of plotlyGlobalStyle.sheet.cssRules) {
					plotlyStyleClone.sheet.insertRule(rule.cssText);
				}
			}
	}

	const main_script = load_bokeh();

	let plugin_scripts = [];
	// Bokeh

	const resolves = [];
	const bokehPromises = Array(5)
		.fill(0)
		.map((_, i) => createPromise(i));

	const initializeBokeh = (index) => {
		if (value && value["type"] == "bokeh") {
			resolves[index]();
		}
	};

	function createPromise(index) {
		return new Promise((resolve, reject) => {
			resolves[index] = resolve;
		});
	}

	function handleBokehLoaded() {
		initializeBokeh(0);
		plugin_scripts = load_plugins();
	}

	Promise.all(bokehPromises).then(() => {
		let plotObj = JSON.parse(value["plot"]);
		window.Bokeh.embed.embed_item(plotObj, "bokehDiv");
	});

	afterUpdate(() => {
		if (value && value["type"] == "plotly") {
			load_plotly_css();
			let plotObj = JSON.parse(value["plot"]);
			plotObj.layout.title ? plotObj.layout.margin = {autoexpand: true} : plotObj.layout.margin = {l: 0, r: 0, b: 0, t: 0};
			Plotly.react(plotDiv, plotObj);
		} else if (value && value["type"] == "bokeh") {
			document.getElementById("bokehDiv").innerHTML = "";
			let plotObj = JSON.parse(value["plot"]);
			window.Bokeh.embed.embed_item(plotObj, "bokehDiv");
		}
	});

	onDestroy(() => {
		if (main_script in document.children) {
			document.removeChild(main_script);
			plugin_scripts.forEach((child) => document.removeChild(child));
		}
	});
</script>

{#if value && value["type"] == "plotly"}
	<div bind:this={plotDiv} />
{:else if value && value["type"] == "bokeh"}
	<div id="bokehDiv" />
{:else if value && value['type'] == "altair"}
	<div class="flex justify-center items-center w-full h-full">
	<Vega spec={spec} />
	</div>
{:else if value && value["type"] == "matplotlib"}
	<div class="output-image w-full flex justify-center items-center relative">
		<!-- svelte-ignore a11y-missing-attribute -->
		<img class="w-full max-h-[30rem] object-contain" src={value["plot"]} />
	</div>
{:else}
	<div class="h-full min-h-[15rem] flex justify-center items-center">
		<div class="h-5 dark:text-white opacity-50"><PlotIcon /></div>
	</div>
{/if}
