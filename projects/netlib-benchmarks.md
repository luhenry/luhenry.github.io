---
title: "Netlib Benchmarks"
layout: default
---

<label for="tag">Tag:</label>
<select name="tag" id="tag">
</select>

<div id="graphs"></div>

<style>
  .wrapper {
    max-width: 100%;
  }
</style>

<script src='https://cdn.plot.ly/plotly-latest.min.js'></script>

<script type="module">
  //FIXME: this import doesn't work, it fails to load es5-ext for some reasons
  // import plotlyJs from 'https://cdn.skypack.dev/plotly.js';
  import { Octokit } from 'https://cdn.skypack.dev/octokit';

  const owner = "luhenry";
  const repo = "netlib";

  const octokit = new Octokit({});

  const tagE = document.getElementById('tag');
  const graphsE = document.getElementById('graphs');

  function isAssetAResult(asset) {
    return asset.name.match(/jmh\-results\-.+\.json/) != null;
  }

  async function* getReleases() {
    for await (const response of octokit.paginate.iterator(octokit.rest.repos.listReleases, { owner, repo })) {
      for (const release of response.data) {
        yield release;
      }
    }
  }

  async function getReleaseByTag(tag) {
    return await octokit.rest.repos.getReleaseByTag({owner, repo, tag })
  }

  async function* getRunsForRelease(release) {
    for (const asset of release.assets) {
      if (!isAssetAResult(asset)) {
        continue;
      }

      const content = await fetch("https://api.ludovic.dev/" + owner + "/" + repo + "/releases/download/" + release.tag_name + "/" + asset.name, {
          headers: {
            'Accept': 'application/octet-stream'
          },
          timeout: 10000,
        }).then(function(response) {
          return response.json();
        });

      for (const run of content) {
        yield run;
      }
    }
  }

  window.onload = async function() {
    var first = true;
    for await (const release of getReleases()) {
      var opt = document.createElement('option');
      opt.value = release.tag_name;
      opt.text = release.tag_name;
      if (release.assets.filter(a => isAssetAResult(a)).length == 0) {
        opt.text += ' (no assets)';
      }
      tagE.appendChild(opt);
      if (first) {
        tagE.value = release.tag_name;
        tagE.dispatchEvent(new Event('change'));
        first = false;
      }
    }
  };

  document.getElementById('tag').onchange = async function() {
    graphsE.innerHTML = '';

    const tag = tagE.options[tagE.selectedIndex].value;
    if (tag === "") {
      console.log("no release selected");
      return;
    }

    const release = (await getReleaseByTag(tag)).data;
    console.log(release);
    if (release.assets.filter(a => isAssetAResult(a)).length == 0) {
      console.log("the release has no assets");
      return;
    }

    var data = new Map();

    for await (const run of getRunsForRelease(release)) {
      if (run.jdkVersion === undefined) {
        console.log("can't parse run, unknown jdkVersion");
        continue;
      }
      const jdkVersionArray = run.jdkVersion.split('.');
      const jdkVersion = parseInt(jdkVersionArray[0]) > 1 ?
                          parseInt(jdkVersionArray[0]) :
                          parseInt(jdkVersionArray[1])

      if (run.params.implementation === undefined) {
        console.log("can't parse run, unknown implementation");
        continue;
      }
      const implementation = run.params.implementation;

      if (run.benchmark === undefined) {
        console.log("can't parse run, unknown benchmark");
        continue;
      }
      const benchmark = run.benchmark.replace(/^dev\.ludovic\.netlib\.benchmarks\.(blas\.l[1-3]|lapack|arpack)\./, '')
                                     .replace(/Benchmark\.(blas|lapack|arpack)$/, '')
                          + '(' + Object.keys(run.params).filter(k => k != 'implementation').map(k => `${k}: ${run.params[k]}`).join(", ") + ')'

      if (run.primaryMetric.score === undefined) {
        console.log("can't parse run, unknown score");
        continue;
      }
      const score = run.primaryMetric.score;

      if (run.primaryMetric.scoreError === undefined) {
        console.log("can't parse run, unknown scoreError");
        continue;
      }
      const scoreError = run.primaryMetric.scoreError;

      if (!data.has(jdkVersion)) {
        data.set(jdkVersion, new Map());
      }
      if (!data.get(jdkVersion).has(implementation)) {
        data.get(jdkVersion).set(implementation, {
          x: [], y: [], yerror: [], ynorm: [], yerrornorm: []
        });
      }
      data.get(jdkVersion).get(implementation).x.push(benchmark);
      data.get(jdkVersion).get(implementation).y.push(score);
      data.get(jdkVersion).get(implementation).yerror.push(scoreError);
    }

    const jdkVersions = Array.from(data.keys()).sort((a, b) => a - b);
    for (const jdkVersion of jdkVersions) {
      const f2j = data.get(jdkVersion).get("f2j");
      for (const implementation of data.get(jdkVersion).keys()) {
        const results = data.get(jdkVersion).get(implementation);
        for (var i = 0; i < results.x.length; i++) {
          //FIXME: assert results.x[i] == f2j.x[i]
          results.ynorm[i] = results.y[i] / f2j.y[i];
          results.yerrornorm[i] = results.yerror[i] / f2j.y[i];
        }
      }
    }

    const colors = {
      'f2j': 'red',
      'java': 'green',
      'native': 'blue',
      // old implementations
      'vector': 'yellow',
    };

    var plotlyData = [];

    var yaxis = 1;
    for (const jdkVersion of jdkVersions) {
      for (const implementation of data.get(jdkVersion).keys()) {
        plotlyData.push({
          type: 'bar',
          legendgroup: implementation,
          name: implementation,
          marker: { 'color': colors[implementation], },
          showlegend: false,
          yaxis: `y${yaxis}`,
          x: data.get(jdkVersion).get(implementation).x,
          y: data.get(jdkVersion).get(implementation).ynorm,
          error_y: {
            type: 'data',
            array: data.get(jdkVersion).get(implementation).yerrornorm,
            visible: true,
          },
          text: data.get(jdkVersion).get(implementation).y.map((t, i) => `score: ${t} +/- ${data.get(jdkVersion).get(implementation).yerror[i]}`),
        });
      }
      yaxis += 1;
    }

    const plotlyLayout = {
      height: 1200,
      grid: {
        rows: 3,
        columns: 1
      },
      xaxis: {
        automargin: true,
        tickangle: 45
      },
    };
    var i = 1;
    for (const jdkVersion of jdkVersions) {
      plotlyLayout[`yaxis${i}`] = {
        title: jdkVersion,
        rangemode: "tozero",
        range: [0, 3],
      }
      i += 1;
    }

    Plotly.newPlot(graphsE.id, plotlyData, plotlyLayout, { responsive: true });
  };
</script>