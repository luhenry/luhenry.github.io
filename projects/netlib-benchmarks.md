---
title: "Netlib Benchmarks"
layout: default
---

<label for="tag">Tag:</label>
<select name="tag" id="tag">
  <option value="" selected></option>
</select>

<div id="graphs"></div>

<style>
  .wrapper {
    max-width: 100%;
  }
</style>

<script src='https://cdn.plot.ly/plotly-latest.min.js'></script>

<script type="module">
  import { Octokit } from "https://cdn.skypack.dev/@octokit/rest";

  const owner = "luhenry";
  const repo = "netlib";

  const octokit = new Octokit({});

  const tagE = document.getElementById('tag');
  const graphsE = document.getElementById('graphs')

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
      if (asset.name.match(/jmh\-results\-.+\.json/) == null) {
        continue;
      }

      const content = await fetch("https://netlib-website.herokuapp.com/" + owner + "/" + repo + "/releases/download/" + release.tag_name + "/" + asset.name, {
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
    for await (const release of getReleases()) {
      var opt = document.createElement('option');
      opt.value = release.tag_name;
      opt.text = release.tag_name;
      tagE.appendChild(opt);
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
    if (release.assets.length == 0) {
      console.log("the release has no assets");
      return;
    }

    var data = new Map();

    for await (const run of getRunsForRelease(release)) {
      const jdkVersionRaw = run.jdkVersion;
      if (jdkVersionRaw === undefined) {
        console.log("can't parse run, unknown jdkVersion");
        continue;
      }
      const jdkVersionRawA = jdkVersionRaw.split('.');
      const jdkVersion = parseInt(jdkVersionRawA[0]) > 1 ?
                          parseInt(jdkVersionRawA[0]) :
                          parseInt(jdkVersionRawA[1])

      const implementation = run.params.implementation;
      if (implementation === undefined) {
        console.log("can't parse run, unknown implementation");
        continue;
      }

      const benchmarkRaw = run.benchmark;
      if (benchmarkRaw === undefined) {
        console.log("can't parse run, unknown benchmark");
        continue;
      }
      const benchmark = benchmarkRaw.replace(/^dev\.ludovic\.netlib\.benchmarks\.(blas\.l[1-3]|lapack|arpack)\./, '').replace(/Benchmark\.(blas|lapack|arpack)$/, '') + '(' + Object.keys(run.params).filter(k => k != 'implementation').map(k => `${k}: ${run.params[k]}`).join(", ") + ')'

      const score = run.primaryMetric.score;
      if (score === undefined) {
        console.log("can't parse run, unknown score");
        continue;
      }

      if (!data.has(jdkVersion)) {
        data.set(jdkVersion, new Map());
      }
      if (!data.get(jdkVersion).has(implementation)) {
        data.get(jdkVersion).set(implementation, {
          x: [], y: [], ynorm: []
        });
      }
      data.get(jdkVersion).get(implementation).x.push(benchmark);
      data.get(jdkVersion).get(implementation).y.push(score);
    }

    for (const jdkVersion of data.keys()) {
      const f2j = data.get(jdkVersion).get("f2j");
      for (const implementation of data.get(jdkVersion).keys()) {
        const results = data.get(jdkVersion).get(implementation);
        for (var i = 0; i < results.x.length; i++) {
          //FIXME: assert results.x[i] == f2j.x[i]
          results.ynorm[i] = results.y[i] / f2j.y[i];
        }
      }
    }

    for (const jdkVersion of data.keys()) {
      const graph = document.createElement('div');
      graph.id = 'graph-' + jdkVersion;
      graphsE.appendChild(graph);

      var plotlyData = [];
      for (const implementation of data.get(jdkVersion).keys()) {
        plotlyData.push({
          type: 'bar',
          name: implementation,
          x: data.get(jdkVersion).get(implementation).x,
          y: data.get(jdkVersion).get(implementation).ynorm,
        });
      }

      const plotlyLayout = {
        barmode: 'group',
        title: jdkVersion,
        // width: 2048 * 2,
        height: 1200,
        xaxis: {
          // zeroline: true,
          automargin: true,
          tickangle: 45
        },
        yaxis: {
          range: [0, 5],
        }
      };

      Plotly.newPlot(graph.id, plotlyData, plotlyLayout, { responsive: true });
    }
  };
</script>