<script>
  import directory from '../data/directory.json';

  import { onMount } from 'svelte';
  import jqx from 'jquery';

  import maplibregl from 'maplibre-gl';
  import MaplibreGeocoder from '@maplibre/maplibre-gl-geocoder';
  import 'maplibre-gl/dist/maplibre-gl.css';
  import '@maplibre/maplibre-gl-geocoder/dist/maplibre-gl-geocoder.css';

  import { Protocol } from 'pmtiles';
  import * as turf from '@turf/turf';

  import mn from '../data/mn.json';
  import mask from '../data/mn-mask.json';
  import basemapStyle from '../data/strib-basemap.json';

  // Register pmtiles:// protocol for MapLibre.
  // This is required if your imported style JSON references PMTiles URLs.
  const protocol = new Protocol();
  maplibregl.addProtocol('pmtiles', protocol.tile);

  let raceData = directory.races;

  /********** CANDIDATE / LEGEND DATA **********/
  // Single source of truth for each primary race: who maps to which base color.
  // Each base color expands into a 5-step light->dark ramp (see buildRamp),
  // used for BOTH the MapLibre fill-color expression and the legend swatches.

  const gopCandidates = [
    { winner: 'Lisa Demuth', color: '#fdbf6f' },
    { winner: 'Kendall Qualls', color: '#e31a1c' },
    { winner: 'Lisa Demuth', color: '#fb9a99' },
    { winner: 'Royce White', color: '#fdbf6f' },
    { winner: 'Michele Tafoya', color: '#e31a1c' },
    { winner: 'Tom Weiler', color: '#fb9a99' },
    { winner: 'Adam Schwarze', color: '#fb8072' },
    { winner: 'Patrick Munro', color: '#888888' },
    { winner: 'Joyce Lacey', color: '#888888' }
  ];

  const dflCandidates = [
    { winner: 'Amy Klobuchar', color: '#80b1d3' },
    { winner: 'Ole Savior', color: '#ffff99' },
    { winner: 'Peggy Flanagan', color: '#b3de69' },
    { winner: 'Angie Craig', color: '#bc80bd' },
    { winner: 'Billy Nord', color: '#888888' },
    { winner: 'George H. Kalberer', color: '#888888' },
    { winner: 'Kurt Michael Anderson', color: '#888888' }
  ];

  // Flat name→color lookup combining both primaries, used by the tooltip
  // to color the dot swatch to match the map's candidate color.
  const candidateColorMap = new Map(
    [...gopCandidates, ...dflCandidates].map(({ winner, color }) => [winner, color])
  );

  // Margin thresholds for the 5-step ramp. Tweak these to match how your
  // winner_margin values are distributed (e.g. percentage points).
  const RAMP_BREAKS = [1, 20, 40, 60, 80];

  /********** COLOR RAMP HELPERS **********/

  function hexToRgb(hex) {
    const h = hex.replace('#', '');
    return [parseInt(h.slice(0, 2), 16), parseInt(h.slice(2, 4), 16), parseInt(h.slice(4, 6), 16)];
  }

  function rgbToHex(rgb) {
    return (
      '#' +
      rgb
        .map((v) => Math.max(0, Math.min(255, Math.round(v))).toString(16).padStart(2, '0'))
        .join('')
    );
  }

  function mixColors(hexA, hexB, t) {
    const a = hexToRgb(hexA);
    const b = hexToRgb(hexB);
    return rgbToHex(a.map((v, i) => v + (b[i] - v) * t));
  }

  // Builds a 5-color light -> dark ramp from a single base color.
  // Index 0 = lightest (weak margin), index 4 = darkest (strong margin).
  function buildRamp(baseColor) {
    return [
      mixColors('#ffffff', baseColor, 0.25),
      mixColors('#ffffff', baseColor, 0.55),
      baseColor,
      mixColors(baseColor, '#000000', 0.2),
      mixColors(baseColor, '#000000', 0.4)
    ];
  }

  // Builds a MapLibre 'case' expression from a candidate list, where each
  // candidate's flat color becomes a 5-step margin-based ramp.
  function buildShadesExpr(candidates, tieColor = '#cfcfcf', defaultColor = '#f0f0f0', noDataColor = '#ffffff') {
    const expr = ['case', ['==', ['get', 'winner_margin'], null], noDataColor];

    for (const { winner, color } of candidates) {
      const ramp = buildRamp(color);
      const stepExpr = ['step', ['get', 'winner_margin'], tieColor];
      RAMP_BREAKS.forEach((breakpoint, i) => {
        stepExpr.push(breakpoint, ramp[i]);
      });

      expr.push(['==', ['get', 'winner'], winner]);
      expr.push(stepExpr);
    }

    expr.push(defaultColor);
    return expr;
  }

  /********** DYNAMIC LEGEND (winners-on-map only) **********/

  // Pulls the distinct 'winner' values that actually appear in the precinct
  // data, so the legend only lists candidates who won at least one precinct.
  async function getDistinctWinners(dataSource) {
    const winners = new Set();

    try {
      const res = await fetch(`./data/${dataSource}`);
      const geojson = await res.json();

      for (const feature of geojson.features || []) {
        const w = feature.properties && feature.properties.winner;
        if (w) winners.add(w);
      }
    } catch (e) {
      console.error('Could not load precinct data for legend filtering:', e);
    }

    return winners;
  }

  // For ticket-style winner names ("Tim Walz and Peggy Flanagan"), show only
  // the first person listed. Also strips a trailing semicolon, which shows
  // up on at least one name in the source data ("...Matt Birk;").
  function firstNameOnly(name) {
    const idx = name.indexOf(' and ');
    const first = idx !== -1 ? name.slice(0, idx) : name;
    return first.trim().replace(/;$/, '');
  }

  // Renders one vertical row per candidate: name on top, 5-swatch ramp below.
  // The full winner name (including running mate) is kept in the title
  // attribute as a hover tooltip, even though the visible text is truncated.
  function renderCandidateLegend(targetSelector, candidates) {
    const rows = candidates
      .map(({ winner, ramp }) => {
        const swatches = ramp.map((c) => `<span style="background-color: ${c};"></span>`).join('');
        return `
          <div class="cand-row">
            <div class="cand-name" title="${winner}">${firstNameOnly(winner)}</div>
            <div class="cand-ramp">${swatches}</div>
          </div>
        `;
      })
      .join('');

    jqx(`${targetSelector} .cand-list`).html(rows);
  }

  // Fetches the actual winners on the map, filters the full candidate list
  // down to just those, builds each one's ramp, and injects the legend rows.
  // Fire-and-forget: runs async without blocking the rest of onMount.
  async function populateDynamicLegend(shading, dataSource) {
    if (shading != 4 && shading != 5) return;

    const candidates = shading == 4 ? gopCandidates : dflCandidates;
    const presentWinners = await getDistinctWinners(dataSource);

    const present = candidates
      .filter((c) => presentWinners.has(c.winner))
      .map((c) => ({ winner: c.winner, ramp: buildRamp(c.color) }));

    renderCandidateLegend(`#legend${shading}`, present);
  }

  function makeMap(
    zoom,
    center,
    interactive,
    shading,
    opacity,
    dataSource,
    overSource,
    filter,
    district,
    overlaid,
    office,
    clicky
  ) {
    /********** MAP CONFIG VARIABLES **********/
    let condition = 'mousemove';
    let mclick = false;
    const statecenter = [-94.033266, 46.472926];
    const metrocenter = [-93.344398, 44.971057];
    const metrozoom = 9;
    const statezoom = zoom;

    let bbox = null;
    let mzoom = zoom;

    /********** INITIALIZE MAP **********/
    const map = new maplibregl.Map({
      container: 'map',
      style: basemapStyle,
      center,
      zoom,
      minZoom: 5.5,
      maxZoom: 14,
      maxBounds: [-107.2, 40.88, -78.92, 51.62],
      scrollZoom: false,
      interactive
    });

    /********** GEOCODER CONFIGURATION **********/
    // MapLibre geocoder expects a geocoding API object rather than a Mapbox token.
    // This follows MapLibre's Nominatim example pattern.
    const geocoderApi = {
      forwardGeocode: async (config) => {
        const features = [];

        try {
          const viewbox = '-97.23965108111081,49.384686592055914,-89.4903653503535,43.49926865177651';
          const request =
            `https://nominatim.openstreetmap.org/search` +
            `?q=${encodeURIComponent(config.query)}` +
            `&format=geojson` +
            `&polygon_geojson=0` +
            `&addressdetails=1` +
            `&countrycodes=us` +
            `&bounded=1` +
            `&viewbox=${viewbox}`;

          const response = await fetch(request, {
            headers: {
              Accept: 'application/json'
            }
          });

          const geojson = await response.json();

          for (const feature of geojson.features || []) {
            const bbox = feature.bbox;
            const center = bbox
              ? [
                  bbox[0] + (bbox[2] - bbox[0]) / 2,
                  bbox[1] + (bbox[3] - bbox[1]) / 2
                ]
              : feature.geometry.coordinates;

            features.push({
              type: 'Feature',
              geometry: {
                type: 'Point',
                coordinates: center
              },
              place_name: feature.properties.display_name,
              properties: feature.properties,
              text: feature.properties.display_name,
              place_type: ['place'],
              center
            });
          }
        } catch (e) {
          console.error('Geocoder error:', e);
        }

        return { features };
      }
    };

    const geocoder = new MaplibreGeocoder(geocoderApi, {
      maplibregl,
      marker:  {
        color: '#5bbf48' // light green
      },
      placeholder: 'Search for location...',
      proximity: {
        longitude: center[0],
        latitude: center[1]
      },
      limit: 5
    });

    const geocoderEl = document.getElementById('geocoder');
    if (geocoderEl) {
      geocoderEl.innerHTML = '';
      geocoderEl.appendChild(geocoder.onAdd(map));
    }

    /********** SPECIAL STATE AND METRO RESET BUTTONS **********/
    class HomeReset {
      onAdd(mapInstance) {
        this.map = mapInstance;
        this.container = document.createElement('div');
        this.container.className = 'maplibregl-ctrl my-custom-control maplibregl-ctrl-group statereset';
        const button = this._createButton('maplibregl-ctrl-icon StateFace monitor_button');
        this.container.appendChild(button);
        return this.container;
      }

      onRemove() {
        this.container.parentNode.removeChild(this.container);
        this.map = undefined;
      }

      _createButton(className) {
        const el = window.document.createElement('button');
        el.className = className;
        el.type = 'button';
        el.innerHTML =
          '<img width="15" src="https://static.startribune.com/news/projects/all/20220215-redistrict/build/img/mn.png" alt="mn" />';
        el.addEventListener(
          'click',
          () => {},
          false
        );
        return el;
      }
    }
    const toggleControl = new HomeReset();

    class MetroReset {
      onAdd(mapInstance) {
        this.map = mapInstance;
        this.container = document.createElement('div');
        this.container.className = 'maplibregl-ctrl my-custom-control maplibregl-ctrl-group metroreset';
        const button = this._createButton('maplibregl-ctrl-icon StateFace monitor_button');
        this.container.appendChild(button);
        return this.container;
      }

      onRemove() {
        this.container.parentNode.removeChild(this.container);
        this.map = undefined;
      }

      _createButton(className) {
        const el = window.document.createElement('button');
        el.className = className;
        el.type = 'button';
        el.innerHTML =
          '<img width="22" src="https://static.startribune.com/news/projects/all/elex22maps/build/img/metro.png" alt="metro" />';
        el.addEventListener(
          'click',
          () => {},
          false
        );
        return el;
      }
    }
    const toggleControlM = new MetroReset();

    /********** SETUP BASIC MAP CONTROLS FOR DESKTOP AND MOBILE **********/
    const scale = new maplibregl.ScaleControl({
      maxWidth: 80,
      unit: 'imperial'
    });

    if (clicky == 1 && office != 214) {
      if (/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)) {
        map.keyboard.disable();
        map.dragRotate.disable();
        map.touchZoomRotate.disableRotation();
        map.scrollZoom.disable();
        map.addControl(new maplibregl.NavigationControl({ showCompass: false }), 'bottom-right');
        condition = 'click';
        mclick = true;
      } else {
        map.addControl(scale);
        map.getCanvas().style.cursor = 'pointer';
        map.doubleClickZoom.disable();
        map.addControl(new maplibregl.NavigationControl({ showCompass: false }), 'bottom-right');
      }

      if (interactive == 1) {
        map.addControl(toggleControl, 'bottom-right');
        map.addControl(toggleControlM, 'bottom-right');

        jqx('#map .statereset').on('click', function () {
          const resetZoom = jqx('#map').width() < 520 ? zoom - 1 : zoom;
          map.jumpTo({
            center: statecenter,
            zoom: resetZoom
          });
        });

        jqx('#map .metroreset').on('click', function () {
          map.jumpTo({
            center: metrocenter,
            zoom: metrozoom
          });
        });
      }
    }

    if (office == 214) {
      map.dragPan.disable();
    }

    /********** ADD MAP LAYERS **********/
    map.on('load', function () {
      if (map.getLayer('water')) {
        map.setPaintProperty('water', 'fill-color', '#ffffff');
      }

      // map.addSource('mask', {
      //   type: 'geojson',
      //   data: mask
      // });

      // map.addLayer(
      //   {
      //     id: 'mask',
      //     source: 'mask',
      //     type: 'fill',
      //     paint: {
      //       'fill-color': '#ffffff',
      //       'fill-opacity': 0.6
      //     }
      //   },
      //   map.getLayer('water') ? 'water' : undefined
      // );

      map.addSource('precincts', {
        type: 'geojson',
        data: `./data/${dataSource}`,
        generateId: true
      });

      map.addLayer(
        {
          id: 'precincts',
          source: 'precincts',
          type: 'fill',
          paint: {
            'fill-color': shading,
            'fill-opacity': opacity
          }
        },
        map.getLayer('water') ? 'water' : undefined
      );

      map.addLayer(
        {
          id: 'precinct-lines',
          source: 'precincts',
          type: 'line',
          paint: {
            'line-color': '#ffffff',
            'line-width': 0
          }
        },
        map.getLayer('water') ? 'water' : undefined
      );

      map.addLayer(
        {
          id: 'precincts-l',
          source: 'precincts',
          type: 'fill',
          paint: {
            'fill-color': '#efefef',
            'fill-opacity': ['case', ['boolean', ['feature-state', 'hover'], false], 0, 0.05],
            'fill-outline-color': '#ffffff'
          }
        },
        map.getLayer('water') ? 'water' : undefined
      );

      if (overlaid == 1) {
        map.addSource('overlay', {
          type: 'geojson',
          data: `../src/data/${overSource}`,
          generateId: true
        });

        const overlayBeforeId = map.getLayer('settlement-subdivision-label')
          ? 'settlement-subdivision-label'
          : undefined;

        map.addLayer(
          {
            id: 'overlay',
            source: 'overlay',
            type: 'fill',
            paint: {
              'fill-color': '#ffffff',
              'fill-opacity': ['case', ['boolean', ['feature-state', 'hover'], false], 0.8, 0],
              'fill-outline-color': [
                'case',
                ['boolean', ['feature-state', 'hover'], false],
                '#000000',
                '#ffffff'
              ]
            }
          },
          overlayBeforeId
        );

        map.addLayer(
          {
            id: 'overlay-l',
            source: 'overlay',
            type: 'line',
            paint: {
              'line-color': '#333333',
              'line-width': 0.3
            }
          },
          overlayBeforeId
        );
      }

      if (filter == 2) {
        map.addSource('overlay-f', {
          type: 'geojson',
          data: `./data/${overSource}`,
          generateId: true
        });

        map.addLayer(
          {
            id: 'overlay-f',
            source: 'overlay-f',
            type: 'line',
            paint: {
              'line-color': '#333333',
              'line-width': 0.5
            },
            filter: ['==', 'DISTRICT', district]
          },
          map.getLayer('water') ? 'water' : undefined
        );

        if (map.getLayer('overlay')) {
          map.setFilter('overlay', ['!=', 'DISTRICT', district]);
          map.setPaintProperty('overlay', 'fill-opacity', 0.8);
        }

        if (office > 1 && office < 10) {
          map.setFilter('precincts', ['==', 'CONGDIST', String(district)]);
        } else if (office > 9 && office < 144) {
          map.setFilter('precincts', ['==', 'MNLEGDIST', String(district)]);
        }

        let loaded = 0;

        map.on('sourcedata', function (e) {
          if (e.sourceId === 'overlay-f' && e.isSourceLoaded && loaded === 0) {
            const f = map.queryRenderedFeatures({ layers: ['overlay-f'] });
            if (f.length === 0) return;

            bbox = turf.bbox({
              type: 'FeatureCollection',
              features: f
            });

            map.fitBounds(bbox, { padding: 20 });
            loaded++;
          }
        });
      }

      map.addSource('mn', {
        type: 'geojson',
        data: mn
      });

      map.addLayer(
        {
          id: 'mn',
          source: 'mn',
          type: 'line',
          paint: {
            'line-color': '#999999'
          }
        },
        map.getLayer('water') ? 'water' : undefined
      );

      /********** TOOLTIP AND HOVER EFFECTS **********/
      let hoveredStateId = null;

      function tooltips(layer) {
        const popup = new maplibregl.Popup({
          closeButton: mclick,
          closeOnClick: mclick,
          className:"tooltip"
        });

        map.on(condition, layer, function (e) {
          map.getCanvas().style.cursor = 'default';
          const feature = e.features?.[0];
          if (!feature) return;

          if (e.features.length > 0) {
            if (hoveredStateId !== null) {
              map.setFeatureState({ source: layer, id: hoveredStateId }, { hover: false });
            }
            hoveredStateId = feature.id;
            map.setFeatureState({ source: layer, id: hoveredStateId }, { hover: true });
          }

          const obj = JSON.parse(feature.properties.votes_obj);
          obj.sort((a, b) => b.votes - a.votes);

          let geo = `${feature.properties.COUNTYNAME} County`;
          if (office > 2 && office < 10) {
            geo = `Congressional District ${feature.properties.CONGDIST}`;
          } else if (office >= 10 && office < 145) {
            geo = `House District ${feature.properties.MNLEGDIST}`;
          }

          popup
            .setLngLat(e.lngLat)
            .setHTML(
              `<span class="precinct-name">${feature.properties.PCTNAME}</span>` +
                `<span class="county-name">${geo}</span>` +
                `${tipinfo(obj, office)}`
            )
            .addTo(map);
        });

        map.on('mouseleave', layer, function () {
          map.getCanvas().style.cursor = '';
          popup.remove();

          if (hoveredStateId !== null) {
            map.setFeatureState({ source: layer, id: hoveredStateId }, { hover: false });
          }
          hoveredStateId = null;
        });
      }

      function tipinfo(obj, office) {
        let tipstring =
          '<table class="tableResults"><thead><tr><th></th><th class="cand"> </th><th class="votes">Votes</th><th class="pct">Pct.</th></tr></thead><tbody>';

        let length = 3;
        if (obj.length < 3) length = obj.length;

        for (let i = 0; i < length; i++) {
          let name;
          if (office > 0) {
            name = obj[i].name;
          } else {
            name = String(obj[i].name).substring(0, String(obj[i].name).indexOf('and'));
          }

          if (name === '') name = 'Write-in';

          tipstring +=
            `<tr>` +
            `<td class="${obj[i].party}">` +
              `<span class="dot ${obj[i].party} ${obj[i].name}"` +
              (candidateColorMap.has(obj[i].name)
                ? ` style="background-color: ${candidateColorMap.get(obj[i].name)};"`
                : '') +
              `></span>` +
            `</td>` +
            `<td class="cand">${name}</td>` +
            `<td class="votes">${obj[i].votes}</td>` +
            `<td class="pct">${Number(obj[i].votes_pct).toFixed(1)}%</td>` +
            `</tr>`;
        }

        tipstring += '</tbody></table>';
        return tipstring;
      }

      tooltips('precincts');

      /********** MOBILE ZOOM ADJUSTMENTS **********/
      jqx(document).ready(function () {
        map.resize();

        if (office == 0 && filter == 3) {
          if (jqx('#map').width() < 520) {
            map.zoomTo(mzoom);
          }
        }

        if (office > 2 && filter == 2) {
          let cachedWidth = jqx(window).width();

          if (jqx('#map').width() < 520 && bbox) {
            map.fitBounds(bbox, { padding: 20 });
          }

          jqx(window).resize(function () {
            const newWidth = jqx(window).width();
            if (newWidth !== cachedWidth) {
              cachedWidth = newWidth;
              if (bbox) {
                map.fitBounds(bbox, { padding: 20 });
              }
            }
          });

          map.on('resize', function () {
            const f = map.queryRenderedFeatures({ layers: ['overlay-f'] });
            if (f.length === 0) return;
            const resizeBbox = turf.bbox({
              type: 'FeatureCollection',
              features: f
            });
            map.fitBounds(resizeBbox, { padding: 20 });
          });
        } else {
          map.on('resize', function () {
            const f = map.queryRenderedFeatures({ layers: ['mn'] });
            if (f.length === 0) return;
            const resizeBbox = turf.bbox({
              type: 'FeatureCollection',
              features: f
            });
            map.fitBounds(resizeBbox, { padding: 20 });
          });
        }
      });
    });
  }

  onMount(() => {
    jqx.urlParam = function (name) {
      const results = new RegExp('[\\?&]' + name + '=([^&#]*)').exec(window.location.href);
      if (results != null) {
        return results[1] || 0;
      } else {
        return null;
      }
    };

    /********** DYNAMIC DATA SETTINGS **********/
    const office = jqx.urlParam('office') ?? 0;
    const format = jqx.urlParam('format') ?? 0;
    const overlay = jqx.urlParam('overlay') ?? 0;
    const filter = jqx.urlParam('filter') ?? 0;
    let boundaries = false;

    if (overlay != 0) boundaries = true;

    let dataSource;

    if (format == 0) dataSource = raceData[office].precincts;
    else if (format == 1) dataSource = raceData[office].counties;
    else if (format == 2) dataSource = raceData[office].congress;
    else if (format == 3) dataSource = raceData[office].house;
    else if (format == 4) dataSource = raceData[office].senate;
    else if (format == 5) dataSource = raceData[office].gop;
    else if (format == 5) dataSource = raceData[office].dfl;

    const overSource = raceData[office].overlay;
    const district = raceData[office].district;

    /********** MAP SHADING OPTIONS **********/
    const shading = jqx.urlParam('shading') ?? 0;

    const shades = [];

    shades[0] = [
      'case',
      ['==', ['get', 'winner_margin'], null], '#ffffff',
      ['==', ['get', 'winner'], 'R'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#f4bcbc', 2, '#f4bcbc', 20, '#ea8c8a', 40, '#d95a59', 60, '#b72e2b', 70, '#851414'],
      ['==', ['get', 'winner'], 'DFL'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#c1d6e6', 2, '#c1d6e6', 20, '#89aecb', 40, '#5186b0', 60, '#285e8d', 70, '#003966'],
      ['==', ['get', 'winner'], 'even'],
      ['step', ['get', 'winner_margin'], '#c5c2d9', 1, '#cfcfcf'],
      '#cfcfcf'
    ];

    shades[1] = [
      'case',
      ['==', ['get', 'winner_margin'], null], '#ffffff',
      ['==', ['get', 'winner'], 'R'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#f4bcbc', 2, '#f4bcbc', 20, '#ea8c8a', 40, '#d95a59', 60, '#b72e2b', 80, '#851414'],
      ['==', ['get', 'winner'], 'DFL'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#c1d6e6', 2, '#c1d6e6', 20, '#89aecb', 40, '#5186b0', 60, '#285e8d', 80, '#003966'],
      ['==', ['get', 'winner'], 'even'],
      ['step', ['get', 'winner_margin'], '#c5c2d9', 1, '#cfcfcf'],
      '#cfcfcf'
    ];

    shades[2] = [
      'case',
      ['==', ['get', 'winner_margin'], null], '#ffffff',
      ['==', ['get', 'winner'], 'R'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#b72e2b'],
      ['==', ['get', 'winner'], 'DFL'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#285e8d'],
      ['==', ['get', 'winner'], 'even'],
      ['step', ['get', 'winner_margin'], '#c5c2d9', 1, '#cfcfcf'],
      '#cfcfcf'
    ];

    shades[3] = [
      'case',
      ['==', ['get', 'winner_margin'], null], '#ffffff',
      ['==', ['get', 'winner'], 'NO'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#e37b40'],
      ['==', ['get', 'winner'], 'YES'],
      ['step', ['get', 'winner_margin'], '#cfcfcf', 1, '#47b33e'],
      '#f0f0f0'
    ];

    //'name' is the candidate name to colorize

    // GOP - 5-step margin ramp built from gopCandidates, defined at the top of this file
    shades[4] = buildShadesExpr(gopCandidates);

    // DFL - 5-step margin ramp built from dflCandidates, defined at the top of this file
    shades[5] = buildShadesExpr(dflCandidates);

    const opacities = [];
    opacities[0] = [
      'step',
      ['coalesce', ['number', ['get', 'votes_sqmi'], 0]],
      0.20,
      20, 0.30,
      30, 0.40,
      40, 0.50,
      50, 0.60,
      60, 0.70,
      70, 0.80,
      80, 0.90,
      90, 0.95
    ];
    opacities[1] = 0.8;
    opacities[2] = 1;
    opacities[3] = 0.8;
    opacities[4] = 0.8;
    opacities[5] = 0.8;

    const legendEl = document.querySelector(`#legend${shading}`);
    if (legendEl) legendEl.style.display = 'block';

    // Populate the GOP/DFL legends with only the candidates who actually won
    // at least one precinct in this data source. Fire-and-forget: doesn't
    // block map setup below.
    populateDynamicLegend(shading, dataSource);

    /********** MAP CONFIGURATION SETTINGS **********/
    const interactive = jqx.urlParam('interactive') ?? 1;
    const clicky = jqx.urlParam('clicky') ?? 1;
    const height = jqx.urlParam('height') ?? 600;
    const search = jqx.urlParam('search') ?? 1;

    const centers = [];
    centers[0] = [-94.033266, 46.472926];
    centers[1] = [-93.344398, 44.971057];
    centers[2] = [-93.90781, 45.940497];
    centers[3] = [-94.033266, 46.472926];
    centers[4] = [-93.48077, 45.001034];
    centers[5] = [-92.100487, 46.786671];
    centers[6] = [-94.156517, 45.559292];
    centers[7] = [-96.767822, 46.87381];
    centers[8] = [-94.005592, 44.166119];
    centers[9] = [-92.45887, 44.019329];

    const zooms = [];
    zooms[0] = 5.5;
    zooms[1] = 9;
    zooms[2] = 5.5;
    zooms[3] = 6;
    zooms[4] = 9;
    zooms[5] = 9;
    zooms[6] = 9.2;
    zooms[7] = 9.5;
    zooms[8] = 10.2;
    zooms[9] = 10.2;

    jqx('#map').css('height', `${height}px`);
    if (clicky == 0) jqx('#map').css('pointer-events', 'none');
    if (search == 0) jqx('#geocoder').hide();

    /********** CHATTER CONFIGURATION **********/
    const text = jqx.urlParam('text') ?? 1;
    const title = jqx.urlParam('title') ?? 'title test';
    const chatter = jqx.urlParam('chatter') ?? 'chatter test';

    if (text == 0) jqx('#text').hide();

    jqx('.chartTitle').html(decodeURI(title).replace(/\+/g, ' '));
    jqx('.chatter').html(decodeURI(chatter).replace(/\+/g, ' '));

    /********** RENDER **********/
    makeMap(
      zooms[filter],
      centers[filter],
      interactive,
      shades[shading],
      opacities[shading],
      dataSource,
      overSource,
      filter,
      district,
      overlay,
      office,
      clicky
    );

    return () => {
      maplibregl.removeProtocol('pmtiles');
    };
  });
</script>

<div id="text">
  <div class="chartTitle">TITLE HERE</div>
  <div class="chatter">chatter here</div>
</div>

<div class="map" id="map">
  <div id="geocoder" class="geocoder"></div>

  <div class="legend" id="legend0">
    <strong class="hide">Lead by vote density</strong>
    <div><span>&nbsp;</span><span style="text-align:right;">&larr;</span><span style="text-align:right;">D</span><span>&nbsp;</span><span>R</span><span>&rarr;</span><span>&nbsp;</span></div>
    <div class="strong"><span style="background-color: #003966"></span><span style="background-color: #5186b0"></span><span style="background-color: #c1d6e6"></span><span style="background-color: #ececec"></span><span style="background-color: #f4bcbc"></span><span style="background-color: #d95a59"></span><span style="background-color: #851414"></span> &darr; votes</div>
    <div class="middle"><span style="background-color: #003966"></span><span style="background-color: #5186b0"></span><span style="background-color: #c1d6e6"></span><span style="background-color: #ececec"></span><span style="background-color: #f4bcbc"></span><span style="background-color: #d95a59"></span><span style="background-color: #851414"></span></div>
    <div class="weak"><span style="background-color: #003966"></span><span style="background-color: #5186b0"></span><span style="background-color: #c1d6e6"></span><span style="background-color: #ececec"></span><span style="background-color: #f4bcbc"></span><span style="background-color: #d95a59"></span><span style="background-color: #851414"></span></div>

    <div class="nodata hide"><span style="background-color: #cfcfcf; border: 1px black solid;"></span> Tie/Close</div>
    <div class="nodata hide"><span style="background-color: #ffffff; border: 1px black solid;"></span> No data</div>
  </div>

  <div class="legend" id="legend1">
    <strong>Lead margin</strong>
    <div><span>&nbsp;</span><span style="text-align:right;">&larr;</span><span style="text-align:right;">D</span><span>&nbsp;</span><span>R</span><span>&rarr;</span><span>&nbsp;</span></div>
    <div class="strong"><span style="background-color: #003966"></span><span style="background-color: #5186b0"></span><span style="background-color: #c1d6e6"></span><span style="background-color: #ececec"></span><span style="background-color: #f4bcbc"></span><span style="background-color: #d95a59"></span><span style="background-color: #851414"></span></div>

    <div class="nodata"><span style="background-color: #cfcfcf; border: 1px black solid;"></span> Tie/Close</div>
    <div class="nodata"><span style="background-color: #ffffff; border: 1px black solid;"></span> No data</div>
  </div>

  <div class="legend" id="legend2">
    <strong>Leader by party</strong>
    <div><span style="text-align:right;">D</span><span>&nbsp;</span><span>R</span></div>
    <div class="strong"><span style="background-color: #003966"></span><span style="background-color: #ececec"></span><span style="background-color: #851414"></span></div>

    <div class="nodata"><span style="background-color: #cfcfcf; border: 1px black solid;"></span> Tie/Close</div>
    <div class="nodata"><span style="background-color: #ffffff; border: 1px black solid;"></span> No data</div>
  </div>

  <div class="legend" id="legend3">
    <strong>Lead by choice</strong>
    <div><span style="text-align:right;">Yes</span><span>&nbsp;</span><span>No</span></div>
    <div class="strong"><span style="background-color: #47b33e"></span><span style="background-color: #ececec"></span><span style="background-color: #e37b40"></span></div>

    <div class="nodata"><span style="background-color: #cfcfcf; border: 1px black solid;"></span> Tie/Close</div>
    <div class="nodata"><span style="background-color: #ffffff; border: 1px black solid;"></span> No data</div>
  </div>

  <!-- GOP PRIMARY -->
  <div class="legend" id="legend4">
    <strong>Leading candidate by margin</strong>
    <div class="cand-list"></div>

    <div class="nodata"><span style="background-color: #cfcfcf; border: 1px black solid;"></span> Tie/Close</div>
    <div class="nodata"><span style="background-color: #ffffff; border: 1px black solid;"></span> No data</div>
  </div>

  <!-- DFL PRIMARY -->
  <div class="legend" id="legend5">
    <strong>Leading candidate by margin</strong>
    <div class="cand-list"></div>

    <div class="nodata"><span style="background-color: #cfcfcf; border: 1px black solid;"></span> Tie/Close</div>
    <div class="nodata"><span style="background-color: #ffffff; border: 1px black solid;"></span> No data</div>
  </div>
</div>

<div class="noteline">
  <em>Vote totals are preliminary and unofficial until canvassed and certified.</em>
</div>

<div class="dataline">
  Source:
  <a href="https://www.sos.state.mn.us/elections-voting/election-results" target="_new">Minnesota Secretary of State</a>,
  <a href="https://gisdata.mn.gov/group/8c21f853-3cdd-4be6-9020-e22ae192dcb4?organization=us-mn-state-lcc&groups=boundaries" target="_new">Minnesota Geospatial Commons</a>,
  <a href="https://protomaps.com/" target="_new">PMTiles/Protomaps</a>,
  <a href="https://www.openstreetmap.org/about/" target="_new">OpenStreetMap</a>
  • Map: Jeff Hargarten and Jake Steinberg, The Minnesota Star Tribune
</div>

<style>
  /*
    These rules target .cand-list/.cand-row/.cand-name/.cand-ramp with :global()
    because that markup is injected at runtime via jQuery .html() (renderCandidateLegend),
    not compiled from this component's template. Svelte's scoped-style hashing only
    reaches elements present in the template at compile time, so without :global()
    these rules would silently never match the injected rows.
  */

  :global(.monitor_button) {
    display: flex !important;
    align-items: center;
    justify-content: center;
  }

  :global(.cand-list) {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin: 4px 0;
  }

  :global(.cand-row) {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  :global(.cand-name) {
    font-size: 1em;
    line-height: 1.15;
  }

  :global(.cand-ramp) {
    display: flex;
  }

  :global(.cand-ramp span) {
    display: inline-block;
    width: 18px;
    height: 10px;
  }

.dataline {
    max-width: 100%;
    font-family: graphik-condensed-medium !important;
    font-size: 11px;
    font-weight: 400;
    color: #888 !important;
    margin-bottom: 2px !important;
}

.noteline {
    max-width: 100%;
    font-family: graphik-condensed-medium !important;
    font-size: 12px;
    font-weight: 400;
    color: #656565 !important;
    margin-top: 16px !important;
    margin-bottom: 3px;
}

.maplibregl-map{
    font:12px/20px graphik-regular;
    overflow:hidden;
    position:relative;
    -webkit-tap-highlight-color:rgba(0,0,0,0)
}
.maplibregl-canvas{
    position:absolute;
    left:0;
    top:0
}
.maplibregl-map:-webkit-full-screen{
    width:100%;
    height:100%
}
.maplibregl-canary{
    background-color:salmon
}
.maplibregl-canvas-container.maplibregl-interactive,.maplibregl-ctrl-group button.maplibregl-ctrl-compass{
    cursor:grab;
    -webkit-user-select:none;
    user-select:none
}
.maplibregl-canvas-container.maplibregl-interactive.maplibregl-track-pointer{
    cursor:pointer
}
.maplibregl-canvas-container.maplibregl-interactive:active,.maplibregl-ctrl-group button.maplibregl-ctrl-compass:active{
    cursor:grabbing
}
.maplibregl-canvas-container.maplibregl-touch-zoom-rotate,.maplibregl-canvas-container.maplibregl-touch-zoom-rotate .maplibregl-canvas{
    touch-action:pan-x pan-y
}
.maplibregl-canvas-container.maplibregl-touch-drag-pan,.maplibregl-canvas-container.maplibregl-touch-drag-pan .maplibregl-canvas{
    touch-action:pinch-zoom
}
.maplibregl-canvas-container.maplibregl-touch-zoom-rotate.maplibregl-touch-drag-pan,.maplibregl-canvas-container.maplibregl-touch-zoom-rotate.maplibregl-touch-drag-pan .maplibregl-canvas{
    touch-action:none
}
.maplibregl-ctrl-bottom-left,.maplibregl-ctrl-bottom-right,.maplibregl-ctrl-top-left,.maplibregl-ctrl-top-right{
    position:absolute;
    pointer-events:none;
    z-index:2
}
.maplibregl-ctrl-top-left{
    top:0;
    left:0
}
.maplibregl-ctrl-top-right{
    top:0;
    right:0
}
.maplibregl-ctrl-bottom-left{
    bottom:0;
    left:0
}
.maplibregl-ctrl-bottom-right{
    right:0;
    bottom:0
}
.maplibregl-ctrl{
    clear:both;
    pointer-events:auto;
    transform:translate(0)
}
.maplibregl-ctrl-top-left .maplibregl-ctrl{
    margin:10px 0 0 10px;
    float:left
}
.maplibregl-ctrl-top-right .maplibregl-ctrl{
    margin:10px 10px 0 0;
    float:right
}
.maplibregl-ctrl-bottom-left .maplibregl-ctrl{
    margin:0 0 10px 10px;
    float:left
}
.maplibregl-ctrl-bottom-right .maplibregl-ctrl{
    margin:0 10px 10px 0;
    float:right
}
.maplibregl-ctrl-group{
    border-radius:4px;
    background:#fff
}
.maplibregl-ctrl-group:not(:empty){
    box-shadow:0 0 0 2px rgba(0,0,0,.1)
}
@media (-ms-high-contrast:active){
    .maplibregl-ctrl-group:not(:empty){
        box-shadow:0 0 0 2px ButtonText
    }
}
.maplibregl-ctrl-group button{
    width:29px;
    height:29px;
    display:block;
    padding:0;
    outline:none;
    text-align:center;
    border:0;
    box-sizing:border-box;
    background-color:transparent;
    cursor:pointer;
    overflow:hidden
}
.maplibregl-ctrl-group button+button{
    border-top:1px solid #ddd
}
.maplibregl-ctrl button .maplibregl-ctrl-icon{
    display:block;
    width:100%;
    height:100%;
    background-repeat:no-repeat;
    background-position:50%
}
@media (-ms-high-contrast:active){
    .maplibregl-ctrl-icon{
        background-color:transparent
    }
    .maplibregl-ctrl-group button+button{
        border-top:1px solid ButtonText
    }
}
.maplibregl-ctrl-attrib-button:focus,.maplibregl-ctrl-group button:focus{
    box-shadow:0 0 2px 2px #0096ff
}
.maplibregl-ctrl button:disabled{
    cursor:not-allowed
}
.maplibregl-ctrl button:disabled .maplibregl-ctrl-icon{
    opacity:.25
}
.maplibregl-ctrl button:not(:disabled):hover{
    background-color:rgba(0,0,0,.05)
}
.maplibregl-ctrl-group button:focus:focus-visible{
    box-shadow:0 0 2px 2px #0096ff
}
.maplibregl-ctrl-group button:focus:not(:focus-visible){
    box-shadow:none
}
.maplibregl-ctrl-group button:focus:first-child{
    border-radius:4px 4px 0 0
}
.maplibregl-ctrl-group button:focus:last-child{
    border-radius:0 0 4px 4px
}
.maplibregl-ctrl-group button:focus:only-child{
    border-radius:inherit
}
.maplibregl-ctrl button.maplibregl-ctrl-zoom-out .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23333'%3E %3Cpath d='M10 13c-.75 0-1.5.75-1.5 1.5S9.25 16 10 16h9c.75 0 1.5-.75 1.5-1.5S19.75 13 19 13h-9z'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-zoom-in .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23333'%3E %3Cpath d='M14.5 8.5c-.75 0-1.5.75-1.5 1.5v3h-3c-.75 0-1.5.75-1.5 1.5S9.25 16 10 16h3v3c0 .75.75 1.5 1.5 1.5S16 19.75 16 19v-3h3c.75 0 1.5-.75 1.5-1.5S19.75 13 19 13h-3v-3c0-.75-.75-1.5-1.5-1.5z'/%3E %3C/svg%3E")
}
@media (-ms-high-contrast:active){
    .maplibregl-ctrl button.maplibregl-ctrl-zoom-out .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23fff'%3E %3Cpath d='M10 13c-.75 0-1.5.75-1.5 1.5S9.25 16 10 16h9c.75 0 1.5-.75 1.5-1.5S19.75 13 19 13h-9z'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-zoom-in .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23fff'%3E %3Cpath d='M14.5 8.5c-.75 0-1.5.75-1.5 1.5v3h-3c-.75 0-1.5.75-1.5 1.5S9.25 16 10 16h3v3c0 .75.75 1.5 1.5 1.5S16 19.75 16 19v-3h3c.75 0 1.5-.75 1.5-1.5S19.75 13 19 13h-3v-3c0-.75-.75-1.5-1.5-1.5z'/%3E %3C/svg%3E")
    }
}
@media (-ms-high-contrast:black-on-white){
    .maplibregl-ctrl button.maplibregl-ctrl-zoom-out .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23000'%3E %3Cpath d='M10 13c-.75 0-1.5.75-1.5 1.5S9.25 16 10 16h9c.75 0 1.5-.75 1.5-1.5S19.75 13 19 13h-9z'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-zoom-in .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23000'%3E %3Cpath d='M14.5 8.5c-.75 0-1.5.75-1.5 1.5v3h-3c-.75 0-1.5.75-1.5 1.5S9.25 16 10 16h3v3c0 .75.75 1.5 1.5 1.5S16 19.75 16 19v-3h3c.75 0 1.5-.75 1.5-1.5S19.75 13 19 13h-3v-3c0-.75-.75-1.5-1.5-1.5z'/%3E %3C/svg%3E")
    }
}
.maplibregl-ctrl button.maplibregl-ctrl-fullscreen .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23333'%3E %3Cpath d='M24 16v5.5c0 1.75-.75 2.5-2.5 2.5H16v-1l3-1.5-4-5.5 1-1 5.5 4 1.5-3h1zM6 16l1.5 3 5.5-4 1 1-4 5.5 3 1.5v1H7.5C5.75 24 5 23.25 5 21.5V16h1zm7-11v1l-3 1.5 4 5.5-1 1-5.5-4L6 13H5V7.5C5 5.75 5.75 5 7.5 5H13zm11 2.5c0-1.75-.75-2.5-2.5-2.5H16v1l3 1.5-4 5.5 1 1 5.5-4 1.5 3h1V7.5z'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-shrink .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg'%3E %3Cpath d='M18.5 16c-1.75 0-2.5.75-2.5 2.5V24h1l1.5-3 5.5 4 1-1-4-5.5 3-1.5v-1h-5.5zM13 18.5c0-1.75-.75-2.5-2.5-2.5H5v1l3 1.5L4 24l1 1 5.5-4 1.5 3h1v-5.5zm3-8c0 1.75.75 2.5 2.5 2.5H24v-1l-3-1.5L25 5l-1-1-5.5 4L17 5h-1v5.5zM10.5 13c1.75 0 2.5-.75 2.5-2.5V5h-1l-1.5 3L5 4 4 5l4 5.5L5 12v1h5.5z'/%3E %3C/svg%3E")
}
@media (-ms-high-contrast:active){
    .maplibregl-ctrl button.maplibregl-ctrl-fullscreen .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23fff'%3E %3Cpath d='M24 16v5.5c0 1.75-.75 2.5-2.5 2.5H16v-1l3-1.5-4-5.5 1-1 5.5 4 1.5-3h1zM6 16l1.5 3 5.5-4 1 1-4 5.5 3 1.5v1H7.5C5.75 24 5 23.25 5 21.5V16h1zm7-11v1l-3 1.5 4 5.5-1 1-5.5-4L6 13H5V7.5C5 5.75 5.75 5 7.5 5H13zm11 2.5c0-1.75-.75-2.5-2.5-2.5H16v1l3 1.5-4 5.5 1 1 5.5-4 1.5 3h1V7.5z'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-shrink .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23fff'%3E %3Cpath d='M18.5 16c-1.75 0-2.5.75-2.5 2.5V24h1l1.5-3 5.5 4 1-1-4-5.5 3-1.5v-1h-5.5zM13 18.5c0-1.75-.75-2.5-2.5-2.5H5v1l3 1.5L4 24l1 1 5.5-4 1.5 3h1v-5.5zm3-8c0 1.75.75 2.5 2.5 2.5H24v-1l-3-1.5L25 5l-1-1-5.5 4L17 5h-1v5.5zM10.5 13c1.75 0 2.5-.75 2.5-2.5V5h-1l-1.5 3L5 4 4 5l4 5.5L5 12v1h5.5z'/%3E %3C/svg%3E")
    }
}
@media (-ms-high-contrast:black-on-white){
    .maplibregl-ctrl button.maplibregl-ctrl-fullscreen .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23000'%3E %3Cpath d='M24 16v5.5c0 1.75-.75 2.5-2.5 2.5H16v-1l3-1.5-4-5.5 1-1 5.5 4 1.5-3h1zM6 16l1.5 3 5.5-4 1 1-4 5.5 3 1.5v1H7.5C5.75 24 5 23.25 5 21.5V16h1zm7-11v1l-3 1.5 4 5.5-1 1-5.5-4L6 13H5V7.5C5 5.75 5.75 5 7.5 5H13zm11 2.5c0-1.75-.75-2.5-2.5-2.5H16v1l3 1.5-4 5.5 1 1 5.5-4 1.5 3h1V7.5z'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-shrink .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23000'%3E %3Cpath d='M18.5 16c-1.75 0-2.5.75-2.5 2.5V24h1l1.5-3 5.5 4 1-1-4-5.5 3-1.5v-1h-5.5zM13 18.5c0-1.75-.75-2.5-2.5-2.5H5v1l3 1.5L4 24l1 1 5.5-4 1.5 3h1v-5.5zm3-8c0 1.75.75 2.5 2.5 2.5H24v-1l-3-1.5L25 5l-1-1-5.5 4L17 5h-1v5.5zM10.5 13c1.75 0 2.5-.75 2.5-2.5V5h-1l-1.5 3L5 4 4 5l4 5.5L5 12v1h5.5z'/%3E %3C/svg%3E")
    }
}
.maplibregl-ctrl button.maplibregl-ctrl-compass .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23333'%3E %3Cpath d='M10.5 14l4-8 4 8h-8z'/%3E %3Cpath id='south' d='M10.5 16l4 8 4-8h-8z' fill='%23ccc'/%3E %3C/svg%3E")
}
@media (-ms-high-contrast:active){
    .maplibregl-ctrl button.maplibregl-ctrl-compass .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23fff'%3E %3Cpath d='M10.5 14l4-8 4 8h-8z'/%3E %3Cpath id='south' d='M10.5 16l4 8 4-8h-8z' fill='%23999'/%3E %3C/svg%3E")
    }
}
@media (-ms-high-contrast:black-on-white){
    .maplibregl-ctrl button.maplibregl-ctrl-compass .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 29 29' xmlns='http://www.w3.org/2000/svg' fill='%23000'%3E %3Cpath d='M10.5 14l4-8 4 8h-8z'/%3E %3Cpath id='south' d='M10.5 16l4 8 4-8h-8z' fill='%23ccc'/%3E %3C/svg%3E")
    }
}
.maplibregl-ctrl button.maplibregl-ctrl-geolocate .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23333'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-geolocate:disabled .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23aaa'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' fill='red'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-active .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%2333b5e5'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-active-error .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23e58978'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-background .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%2333b5e5'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2' display='none'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-background-error .maplibregl-ctrl-icon{
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23e54e33'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2' display='none'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
}
.maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-waiting .maplibregl-ctrl-icon{
    animation:maplibregl-spin 2s linear infinite
}
@media (-ms-high-contrast:active){
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23fff'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate:disabled .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23999'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' fill='red'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-active .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%2333b5e5'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-active-error .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23e58978'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-background .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%2333b5e5'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2' display='none'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate.maplibregl-ctrl-geolocate-background-error .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23e54e33'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2' display='none'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
    }
}
@media (-ms-high-contrast:black-on-white){
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23000'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' display='none'/%3E %3C/svg%3E")
    }
    .maplibregl-ctrl button.maplibregl-ctrl-geolocate:disabled .maplibregl-ctrl-icon{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='29' height='29' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill='%23666'%3E %3Cpath d='M10 4C9 4 9 5 9 5v.1A5 5 0 0 0 5.1 9H5s-1 0-1 1 1 1 1 1h.1A5 5 0 0 0 9 14.9v.1s0 1 1 1 1-1 1-1v-.1a5 5 0 0 0 3.9-3.9h.1s1 0 1-1-1-1-1-1h-.1A5 5 0 0 0 11 5.1V5s0-1-1-1zm0 2.5a3.5 3.5 0 1 1 0 7 3.5 3.5 0 1 1 0-7z'/%3E %3Ccircle id='dot' cx='10' cy='10' r='2'/%3E %3Cpath id='stroke' d='M14 5l1 1-9 9-1-1 9-9z' fill='red'/%3E %3C/svg%3E")
    }
}
@keyframes maplibregl-spin{
    0%{
        transform:rotate(0deg)
    }
    to{
        transform:rotate(1turn)
    }
}
a.maplibregl-ctrl-logo{
    width:88px;
    height:23px;
    margin:0 0 -4px -4px;
    display:block;
    background-repeat:no-repeat;
    cursor:pointer;
    overflow:hidden;
    background-image:url("data:image/svg+xml;charset=utf-8,%3Csvg width='88' height='23' viewBox='0 0 88 23' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' fill-rule='evenodd'%3E %3Cdefs%3E %3Cpath id='logo' d='M11.5 2.25c5.105 0 9.25 4.145 9.25 9.25s-4.145 9.25-9.25 9.25-9.25-4.145-9.25-9.25 4.145-9.25 9.25-9.25zM6.997 15.983c-.051-.338-.828-5.802 2.233-8.873a4.395 4.395 0 013.13-1.28c1.27 0 2.49.51 3.39 1.42.91.9 1.42 2.12 1.42 3.39 0 1.18-.449 2.301-1.28 3.13C12.72 16.93 7 16 7 16l-.003-.017zM15.3 10.5l-2 .8-.8 2-.8-2-2-.8 2-.8.8-2 .8 2 2 .8z'/%3E %3Cpath id='text' d='M50.63 8c.13 0 .23.1.23.23V9c.7-.76 1.7-1.18 2.73-1.18 2.17 0 3.95 1.85 3.95 4.17s-1.77 4.19-3.94 4.19c-1.04 0-2.03-.43-2.74-1.18v3.77c0 .13-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V8.23c0-.12.1-.23.23-.23h1.4zm-3.86.01c.01 0 .01 0 .01-.01.13 0 .22.1.22.22v7.55c0 .12-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V15c-.7.76-1.69 1.19-2.73 1.19-2.17 0-3.94-1.87-3.94-4.19 0-2.32 1.77-4.19 3.94-4.19 1.03 0 2.02.43 2.73 1.18v-.75c0-.12.1-.23.23-.23h1.4zm26.375-.19a4.24 4.24 0 00-4.16 3.29c-.13.59-.13 1.19 0 1.77a4.233 4.233 0 004.17 3.3c2.35 0 4.26-1.87 4.26-4.19 0-2.32-1.9-4.17-4.27-4.17zM60.63 5c.13 0 .23.1.23.23v3.76c.7-.76 1.7-1.18 2.73-1.18 1.88 0 3.45 1.4 3.84 3.28.13.59.13 1.2 0 1.8-.39 1.88-1.96 3.29-3.84 3.29-1.03 0-2.02-.43-2.73-1.18v.77c0 .12-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V5.23c0-.12.1-.23.23-.23h1.4zm-34 11h-1.4c-.13 0-.23-.11-.23-.23V8.22c.01-.13.1-.22.23-.22h1.4c.13 0 .22.11.23.22v.68c.5-.68 1.3-1.09 2.16-1.1h.03c1.09 0 2.09.6 2.6 1.55.45-.95 1.4-1.55 2.44-1.56 1.62 0 2.93 1.25 2.9 2.78l.03 5.2c0 .13-.1.23-.23.23h-1.41c-.13 0-.23-.11-.23-.23v-4.59c0-.98-.74-1.71-1.62-1.71-.8 0-1.46.7-1.59 1.62l.01 4.68c0 .13-.11.23-.23.23h-1.41c-.13 0-.23-.11-.23-.23v-4.59c0-.98-.74-1.71-1.62-1.71-.85 0-1.54.79-1.6 1.8v4.5c0 .13-.1.23-.23.23zm53.615 0h-1.61c-.04 0-.08-.01-.12-.03-.09-.06-.13-.19-.06-.28l2.43-3.71-2.39-3.65a.213.213 0 01-.03-.12c0-.12.09-.21.21-.21h1.61c.13 0 .24.06.3.17l1.41 2.37 1.4-2.37a.34.34 0 01.3-.17h1.6c.04 0 .08.01.12.03.09.06.13.19.06.28l-2.37 3.65 2.43 3.7c0 .05.01.09.01.13 0 .12-.09.21-.21.21h-1.61c-.13 0-.24-.06-.3-.17l-1.44-2.42-1.44 2.42a.34.34 0 01-.3.17zm-7.12-1.49c-1.33 0-2.42-1.12-2.42-2.51 0-1.39 1.08-2.52 2.42-2.52 1.33 0 2.42 1.12 2.42 2.51 0 1.39-1.08 2.51-2.42 2.52zm-19.865 0c-1.32 0-2.39-1.11-2.42-2.48v-.07c.02-1.38 1.09-2.49 2.4-2.49 1.32 0 2.41 1.12 2.41 2.51 0 1.39-1.07 2.52-2.39 2.53zm-8.11-2.48c-.01 1.37-1.09 2.47-2.41 2.47s-2.42-1.12-2.42-2.51c0-1.39 1.08-2.52 2.4-2.52 1.33 0 2.39 1.11 2.41 2.48l.02.08zm18.12 2.47c-1.32 0-2.39-1.11-2.41-2.48v-.06c.02-1.38 1.09-2.48 2.41-2.48s2.42 1.12 2.42 2.51c0 1.39-1.09 2.51-2.42 2.51z'/%3E %3C/defs%3E %3Cmask id='clip'%3E %3Crect x='0' y='0' width='100%25' height='100%25' fill='white'/%3E %3Cuse xlink:href='%23logo'/%3E %3Cuse xlink:href='%23text'/%3E %3C/mask%3E %3Cg id='outline' opacity='0.3' stroke='%23000' stroke-width='3'%3E %3Ccircle mask='url(%23clip)' cx='11.5' cy='11.5' r='9.25'/%3E %3Cuse xlink:href='%23text' mask='url(%23clip)'/%3E %3C/g%3E %3Cg id='fill' opacity='0.9' fill='%23fff'%3E %3Cuse xlink:href='%23logo'/%3E %3Cuse xlink:href='%23text'/%3E %3C/g%3E %3C/svg%3E")
}
a.maplibregl-ctrl-logo.maplibregl-compact{
    width:23px
}
@media (-ms-high-contrast:active){
    a.maplibregl-ctrl-logo{
        background-color:transparent;
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='88' height='23' viewBox='0 0 88 23' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' fill-rule='evenodd'%3E %3Cdefs%3E %3Cpath id='logo' d='M11.5 2.25c5.105 0 9.25 4.145 9.25 9.25s-4.145 9.25-9.25 9.25-9.25-4.145-9.25-9.25 4.145-9.25 9.25-9.25zM6.997 15.983c-.051-.338-.828-5.802 2.233-8.873a4.395 4.395 0 013.13-1.28c1.27 0 2.49.51 3.39 1.42.91.9 1.42 2.12 1.42 3.39 0 1.18-.449 2.301-1.28 3.13C12.72 16.93 7 16 7 16l-.003-.017zM15.3 10.5l-2 .8-.8 2-.8-2-2-.8 2-.8.8-2 .8 2 2 .8z'/%3E %3Cpath id='text' d='M50.63 8c.13 0 .23.1.23.23V9c.7-.76 1.7-1.18 2.73-1.18 2.17 0 3.95 1.85 3.95 4.17s-1.77 4.19-3.94 4.19c-1.04 0-2.03-.43-2.74-1.18v3.77c0 .13-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V8.23c0-.12.1-.23.23-.23h1.4zm-3.86.01c.01 0 .01 0 .01-.01.13 0 .22.1.22.22v7.55c0 .12-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V15c-.7.76-1.69 1.19-2.73 1.19-2.17 0-3.94-1.87-3.94-4.19 0-2.32 1.77-4.19 3.94-4.19 1.03 0 2.02.43 2.73 1.18v-.75c0-.12.1-.23.23-.23h1.4zm26.375-.19a4.24 4.24 0 00-4.16 3.29c-.13.59-.13 1.19 0 1.77a4.233 4.233 0 004.17 3.3c2.35 0 4.26-1.87 4.26-4.19 0-2.32-1.9-4.17-4.27-4.17zM60.63 5c.13 0 .23.1.23.23v3.76c.7-.76 1.7-1.18 2.73-1.18 1.88 0 3.45 1.4 3.84 3.28.13.59.13 1.2 0 1.8-.39 1.88-1.96 3.29-3.84 3.29-1.03 0-2.02-.43-2.73-1.18v.77c0 .12-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V5.23c0-.12.1-.23.23-.23h1.4zm-34 11h-1.4c-.13 0-.23-.11-.23-.23V8.22c.01-.13.1-.22.23-.22h1.4c.13 0 .22.11.23.22v.68c.5-.68 1.3-1.09 2.16-1.1h.03c1.09 0 2.09.6 2.6 1.55.45-.95 1.4-1.55 2.44-1.56 1.62 0 2.93 1.25 2.9 2.78l.03 5.2c0 .13-.1.23-.23.23h-1.41c-.13 0-.23-.11-.23-.23v-4.59c0-.98-.74-1.71-1.62-1.71-.8 0-1.46.7-1.59 1.62l.01 4.68c0 .13-.11.23-.23.23h-1.41c-.13 0-.23-.11-.23-.23v-4.59c0-.98-.74-1.71-1.62-1.71-.85 0-1.54.79-1.6 1.8v4.5c0 .13-.1.23-.23.23zm53.615 0h-1.61c-.04 0-.08-.01-.12-.03-.09-.06-.13-.19-.06-.28l2.43-3.71-2.39-3.65a.213.213 0 01-.03-.12c0-.12.09-.21.21-.21h1.61c.13 0 .24.06.3.17l1.41 2.37 1.4-2.37a.34.34 0 01.3-.17h1.6c.04 0 .08.01.12.03.09.06.13.19.06.28l-2.37 3.65 2.43 3.7c0 .05.01.09.01.13 0 .12-.09.21-.21.21h-1.61c-.13 0-.24-.06-.3-.17l-1.44-2.42-1.44 2.42a.34.34 0 01-.3.17zm-7.12-1.49c-1.33 0-2.42-1.12-2.42-2.51 0-1.39 1.08-2.52 2.42-2.52 1.33 0 2.42 1.12 2.42 2.51 0 1.39-1.08 2.51-2.42 2.52zm-19.865 0c-1.32 0-2.39-1.11-2.42-2.48v-.07c.02-1.38 1.09-2.49 2.4-2.49 1.32 0 2.41 1.12 2.41 2.51 0 1.39-1.07 2.52-2.39 2.53zm-8.11-2.48c-.01 1.37-1.09 2.47-2.41 2.47s-2.42-1.12-2.42-2.51c0-1.39 1.08-2.52 2.4-2.52 1.33 0 2.39 1.11 2.41 2.48l.02.08zm18.12 2.47c-1.32 0-2.39-1.11-2.41-2.48v-.06c.02-1.38 1.09-2.48 2.41-2.48s2.42 1.12 2.42 2.51c0 1.39-1.09 2.51-2.42 2.51z'/%3E %3C/defs%3E %3Cmask id='clip'%3E %3Crect x='0' y='0' width='100%25' height='100%25' fill='white'/%3E %3Cuse xlink:href='%23logo'/%3E %3Cuse xlink:href='%23text'/%3E %3C/mask%3E %3Cg id='outline' opacity='1' stroke='%23000' stroke-width='3'%3E %3Ccircle mask='url(%23clip)' cx='11.5' cy='11.5' r='9.25'/%3E %3Cuse xlink:href='%23text' mask='url(%23clip)'/%3E %3C/g%3E %3Cg id='fill' opacity='1' fill='%23fff'%3E %3Cuse xlink:href='%23logo'/%3E %3Cuse xlink:href='%23text'/%3E %3C/g%3E %3C/svg%3E")
    }
}
@media (-ms-high-contrast:black-on-white){
    a.maplibregl-ctrl-logo{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='88' height='23' viewBox='0 0 88 23' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' fill-rule='evenodd'%3E %3Cdefs%3E %3Cpath id='logo' d='M11.5 2.25c5.105 0 9.25 4.145 9.25 9.25s-4.145 9.25-9.25 9.25-9.25-4.145-9.25-9.25 4.145-9.25 9.25-9.25zM6.997 15.983c-.051-.338-.828-5.802 2.233-8.873a4.395 4.395 0 013.13-1.28c1.27 0 2.49.51 3.39 1.42.91.9 1.42 2.12 1.42 3.39 0 1.18-.449 2.301-1.28 3.13C12.72 16.93 7 16 7 16l-.003-.017zM15.3 10.5l-2 .8-.8 2-.8-2-2-.8 2-.8.8-2 .8 2 2 .8z'/%3E %3Cpath id='text' d='M50.63 8c.13 0 .23.1.23.23V9c.7-.76 1.7-1.18 2.73-1.18 2.17 0 3.95 1.85 3.95 4.17s-1.77 4.19-3.94 4.19c-1.04 0-2.03-.43-2.74-1.18v3.77c0 .13-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V8.23c0-.12.1-.23.23-.23h1.4zm-3.86.01c.01 0 .01 0 .01-.01.13 0 .22.1.22.22v7.55c0 .12-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V15c-.7.76-1.69 1.19-2.73 1.19-2.17 0-3.94-1.87-3.94-4.19 0-2.32 1.77-4.19 3.94-4.19 1.03 0 2.02.43 2.73 1.18v-.75c0-.12.1-.23.23-.23h1.4zm26.375-.19a4.24 4.24 0 00-4.16 3.29c-.13.59-.13 1.19 0 1.77a4.233 4.233 0 004.17 3.3c2.35 0 4.26-1.87 4.26-4.19 0-2.32-1.9-4.17-4.27-4.17zM60.63 5c.13 0 .23.1.23.23v3.76c.7-.76 1.7-1.18 2.73-1.18 1.88 0 3.45 1.4 3.84 3.28.13.59.13 1.2 0 1.8-.39 1.88-1.96 3.29-3.84 3.29-1.03 0-2.02-.43-2.73-1.18v.77c0 .12-.1.23-.23.23h-1.4c-.13 0-.23-.1-.23-.23V5.23c0-.12.1-.23.23-.23h1.4zm-34 11h-1.4c-.13 0-.23-.11-.23-.23V8.22c.01-.13.1-.22.23-.22h1.4c.13 0 .22.11.23.22v.68c.5-.68 1.3-1.09 2.16-1.1h.03c1.09 0 2.09.6 2.6 1.55.45-.95 1.4-1.55 2.44-1.56 1.62 0 2.93 1.25 2.9 2.78l.03 5.2c0 .13-.1.23-.23.23h-1.41c-.13 0-.23-.11-.23-.23v-4.59c0-.98-.74-1.71-1.62-1.71-.8 0-1.46.7-1.59 1.62l.01 4.68c0 .13-.11.23-.23.23h-1.41c-.13 0-.23-.11-.23-.23v-4.59c0-.98-.74-1.71-1.62-1.71-.85 0-1.54.79-1.6 1.8v4.5c0 .13-.1.23-.23.23zm53.615 0h-1.61c-.04 0-.08-.01-.12-.03-.09-.06-.13-.19-.06-.28l2.43-3.71-2.39-3.65a.213.213 0 01-.03-.12c0-.12.09-.21.21-.21h1.61c.13 0 .24.06.3.17l1.41 2.37 1.4-2.37a.34.34 0 01.3-.17h1.6c.04 0 .08.01.12.03.09.06.13.19.06.28l-2.37 3.65 2.43 3.7c0 .05.01.09.01.13 0 .12-.09.21-.21.21h-1.61c-.13 0-.24-.06-.3-.17l-1.44-2.42-1.44 2.42a.34.34 0 01-.3.17zm-7.12-1.49c-1.33 0-2.42-1.12-2.42-2.51 0-1.39 1.08-2.52 2.42-2.52 1.33 0 2.42 1.12 2.42 2.51 0 1.39-1.08 2.51-2.42 2.52zm-19.865 0c-1.32 0-2.39-1.11-2.42-2.48v-.07c.02-1.38 1.09-2.49 2.4-2.49 1.32 0 2.41 1.12 2.41 2.51 0 1.39-1.07 2.52-2.39 2.53zm-8.11-2.48c-.01 1.37-1.09 2.47-2.41 2.47s-2.42-1.12-2.42-2.51c0-1.39 1.08-2.52 2.4-2.52 1.33 0 2.39 1.11 2.41 2.48l.02.08zm18.12 2.47c-1.32 0-2.39-1.11-2.41-2.48v-.06c.02-1.38 1.09-2.48 2.41-2.48s2.42 1.12 2.42 2.51c0 1.39-1.09 2.51-2.42 2.51z'/%3E %3C/defs%3E %3Cmask id='clip'%3E %3Crect x='0' y='0' width='100%25' height='100%25' fill='white'/%3E %3Cuse xlink:href='%23logo'/%3E %3Cuse xlink:href='%23text'/%3E %3C/mask%3E %3Cg id='outline' opacity='1' stroke='%23fff' stroke-width='3' fill='%23fff'%3E %3Ccircle mask='url(%23clip)' cx='11.5' cy='11.5' r='9.25'/%3E %3Cuse xlink:href='%23text' mask='url(%23clip)'/%3E %3C/g%3E %3Cg id='fill' opacity='1' fill='%23000'%3E %3Cuse xlink:href='%23logo'/%3E %3Cuse xlink:href='%23text'/%3E %3C/g%3E %3C/svg%3E")
    }
}
.maplibregl-ctrl.maplibregl-ctrl-attrib{
    padding:0 5px;
    background-color:hsla(0,0%,100%,.5);
    margin:0
}
@media screen{
    .maplibregl-ctrl-attrib.maplibregl-compact{
        min-height:20px;
        padding:2px 24px 2px 0;
        margin:10px;
        position:relative;
        background-color:#fff;
        border-radius:12px
    }
    .maplibregl-ctrl-attrib.maplibregl-compact-show{
        padding:2px 28px 2px 8px;
        visibility:visible
    }
    .maplibregl-ctrl-bottom-left>.maplibregl-ctrl-attrib.maplibregl-compact-show,.maplibregl-ctrl-top-left>.maplibregl-ctrl-attrib.maplibregl-compact-show{
        padding:2px 8px 2px 28px;
        border-radius:12px
    }
    .maplibregl-ctrl-attrib.maplibregl-compact .maplibregl-ctrl-attrib-inner{
        display:none
    }
    .maplibregl-ctrl-attrib-button{
        display:none;
        cursor:pointer;
        position:absolute;
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='24' height='24' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill-rule='evenodd'%3E %3Cpath d='M4 10a6 6 0 1 0 12 0 6 6 0 1 0-12 0m5-3a1 1 0 1 0 2 0 1 1 0 1 0-2 0m0 3a1 1 0 1 1 2 0v3a1 1 0 1 1-2 0'/%3E %3C/svg%3E");
        background-color:hsla(0,0%,100%,.5);
        width:24px;
        height:24px;
        box-sizing:border-box;
        border-radius:12px;
        outline:none;
        top:0;
        right:0;
        border:0
    }
    .maplibregl-ctrl-bottom-left .maplibregl-ctrl-attrib-button,.maplibregl-ctrl-top-left .maplibregl-ctrl-attrib-button{
        left:0
    }
    .maplibregl-ctrl-attrib.maplibregl-compact-show .maplibregl-ctrl-attrib-inner,.maplibregl-ctrl-attrib.maplibregl-compact .maplibregl-ctrl-attrib-button{
        display:block
    }
    .maplibregl-ctrl-attrib.maplibregl-compact-show .maplibregl-ctrl-attrib-button{
        background-color:rgba(0,0,0,.05)
    }
    .maplibregl-ctrl-bottom-right>.maplibregl-ctrl-attrib.maplibregl-compact:after{
        bottom:0;
        right:0
    }
    .maplibregl-ctrl-top-right>.maplibregl-ctrl-attrib.maplibregl-compact:after{
        top:0;
        right:0
    }
    .maplibregl-ctrl-top-left>.maplibregl-ctrl-attrib.maplibregl-compact:after{
        top:0;
        left:0
    }
    .maplibregl-ctrl-bottom-left>.maplibregl-ctrl-attrib.maplibregl-compact:after{
        bottom:0;
        left:0
    }
}
@media screen and (-ms-high-contrast:active){
    .maplibregl-ctrl-attrib.maplibregl-compact:after{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='24' height='24' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill-rule='evenodd' fill='%23fff'%3E %3Cpath d='M4 10a6 6 0 1 0 12 0 6 6 0 1 0-12 0m5-3a1 1 0 1 0 2 0 1 1 0 1 0-2 0m0 3a1 1 0 1 1 2 0v3a1 1 0 1 1-2 0'/%3E %3C/svg%3E")
    }
}
@media screen and (-ms-high-contrast:black-on-white){
    .maplibregl-ctrl-attrib.maplibregl-compact:after{
        background-image:url("data:image/svg+xml;    charset=utf-8,%3Csvg width='24' height='24' viewBox='0 0 20 20' xmlns='http://www.w3.org/2000/svg' fill-rule='evenodd'%3E %3Cpath d='M4 10a6 6 0 1 0 12 0 6 6 0 1 0-12 0m5-3a1 1 0 1 0 2 0 1 1 0 1 0-2 0m0 3a1 1 0 1 1 2 0v3a1 1 0 1 1-2 0'/%3E %3C/svg%3E")
    }
}
.maplibregl-ctrl-attrib a{
    color:rgba(0,0,0,.75);
    text-decoration:none
}
.maplibregl-ctrl-attrib a:hover{
    color:inherit;
    text-decoration:underline
}
.maplibregl-ctrl-attrib .mapbox-improve-map{
    font-weight:700;
    margin-left:2px
}
.maplibregl-attrib-empty{
    display:none
}
.maplibregl-ctrl-scale{
    background-color:hsla(0,0%,100%,.75);
    font-size:10px;
    border:2px solid #333;
    border-top:#333;
    padding:0 5px;
    color:#333;
    box-sizing:border-box
}
.maplibregl-popup{
    position:absolute;
    top:0;
    left:0;
    display:flex;
    will-change:transform;
    pointer-events:none
}
.maplibregl-popup-anchor-top,.maplibregl-popup-anchor-top-left,.maplibregl-popup-anchor-top-right{
    flex-direction:column
}
.maplibregl-popup-anchor-bottom,.maplibregl-popup-anchor-bottom-left,.maplibregl-popup-anchor-bottom-right{
    flex-direction:column-reverse
}
.maplibregl-popup-anchor-left{
    flex-direction:row
}
.maplibregl-popup-anchor-right{
    flex-direction:row-reverse
}
.maplibregl-popup-tip{
    width:0;
    height:0;
    border:10px solid transparent;
    z-index:1
}
.maplibregl-popup-anchor-top .maplibregl-popup-tip{
    align-self:center;
    border-top:none;
    border-bottom-color:#fff
}
.maplibregl-popup-anchor-top-left .maplibregl-popup-tip{
    align-self:flex-start;
    border-top:none;
    border-left:none;
    border-bottom-color:#fff
}
.maplibregl-popup-anchor-top-right .maplibregl-popup-tip{
    align-self:flex-end;
    border-top:none;
    border-right:none;
    border-bottom-color:#fff
}
.maplibregl-popup-anchor-bottom .maplibregl-popup-tip{
    align-self:center;
    border-bottom:none;
    border-top-color:#fff
}
.maplibregl-popup-anchor-bottom-left .maplibregl-popup-tip{
    align-self:flex-start;
    border-bottom:none;
    border-left:none;
    border-top-color:#fff
}
.maplibregl-popup-anchor-bottom-right .maplibregl-popup-tip{
    align-self:flex-end;
    border-bottom:none;
    border-right:none;
    border-top-color:#fff
}
.maplibregl-popup-anchor-left .maplibregl-popup-tip{
    align-self:center;
    border-left:none;
    border-right-color:#fff
}
.maplibregl-popup-anchor-right .maplibregl-popup-tip{
    align-self:center;
    border-right:none;
    border-left-color:#fff
}
.maplibregl-popup-close-button{
    position:absolute;
    right:0;
    top:0;
    border:0;
    border-radius:0 3px 0 0;
    cursor:pointer;
    background-color:transparent
}
.maplibregl-popup-close-button:hover{
    background-color:rgba(0,0,0,.05)
}
.maplibregl-popup-content{
    position:relative;
    background:#fff;
    border-radius:3px;
    box-shadow:0 1px 2px rgba(0,0,0,.1);
    padding:10px 10px 15px;
    pointer-events:auto
}
.maplibregl-popup-anchor-top-left .maplibregl-popup-content{
    border-top-left-radius:0
}
.maplibregl-popup-anchor-top-right .maplibregl-popup-content{
    border-top-right-radius:0
}
.maplibregl-popup-anchor-bottom-left .maplibregl-popup-content{
    border-bottom-left-radius:0
}
.maplibregl-popup-anchor-bottom-right .maplibregl-popup-content{
    border-bottom-right-radius:0
}
.maplibregl-popup-track-pointer{
    display:none
}
.maplibregl-popup-track-pointer *{
    pointer-events:none;
    user-select:none
}
.maplibregl-map:hover .maplibregl-popup-track-pointer{
    display:flex
}
.maplibregl-map:active .maplibregl-popup-track-pointer{
    display:none
}
.maplibregl-marker{
    position:absolute;
    top:0;
    left:0;
    will-change:transform;
    opacity:1;
    transition:opacity .2s
}
.maplibregl-user-location-dot,.maplibregl-user-location-dot:before{
    background-color:#1da1f2;
    width:15px;
    height:15px;
    border-radius:50%
}
.maplibregl-user-location-dot:before{
    content:"";
    position:absolute;
    animation:maplibregl-user-location-dot-pulse 2s infinite
}
.maplibregl-user-location-dot:after{
    border-radius:50%;
    border:2px solid #fff;
    content:"";
    height:19px;
    left:-2px;
    position:absolute;
    top:-2px;
    width:19px;
    box-sizing:border-box;
    box-shadow:0 0 3px rgba(0,0,0,.35)
}
.maplibregl-user-location-show-heading .maplibregl-user-location-heading{
    width:0;
    height:0
}
.maplibregl-user-location-show-heading .maplibregl-user-location-heading:after,.maplibregl-user-location-show-heading .maplibregl-user-location-heading:before{
    content:"";
    border-bottom:7.5px solid #4aa1eb;
    position:absolute
}
.maplibregl-user-location-show-heading .maplibregl-user-location-heading:before{
    border-left:7.5px solid transparent;
    transform:translateY(-28px) skewY(-20deg)
}
.maplibregl-user-location-show-heading .maplibregl-user-location-heading:after{
    border-right:7.5px solid transparent;
    transform:translate(7.5px,-28px) skewY(20deg)
}
@keyframes maplibregl-user-location-dot-pulse{
    0%{
        transform:scale(1);
        opacity:1
    }
    70%{
        transform:scale(3);
        opacity:0
    }
    to{
        transform:scale(1);
        opacity:0
    }
}
.maplibregl-user-location-dot-stale{
    background-color:#aaa
}
.maplibregl-user-location-dot-stale:after{
    display:none
}
.maplibregl-user-location-accuracy-circle{
    background-color:rgba(29,161,242,.2);
    width:1px;
    height:1px;
    border-radius:100%
}
.maplibregl-crosshair,.maplibregl-crosshair .maplibregl-interactive,.maplibregl-crosshair .maplibregl-interactive:active{
    cursor:crosshair
}
.maplibregl-boxzoom{
    position:absolute;
    top:0;
    left:0;
    width:0;
    height:0;
    background:#fff;
    border:2px dotted #202020;
    opacity:.5
}
@media print{
    .mapbox-improve-map{
        display:none
    }
}
.maplibregl-scroll-zoom-blocker,.maplibregl-touch-pan-blocker{
    color:#fff;
    font-family:-apple-system,BlinkMacSystemFont,Segoe UI,Helvetica,Arial,sans-serif;
    justify-content:center;
    text-align:center;
    position:absolute;
    display:flex;
    align-items:center;
    top:0;
    left:0;
    width:100%;
    height:100%;
    background:rgba(0,0,0,.7);
    opacity:0;
    pointer-events:none;
    transition:opacity .75s ease-in-out;
    transition-delay:1s
}
.maplibregl-scroll-zoom-blocker-show,.maplibregl-touch-pan-blocker-show{
    opacity:1;
    transition:opacity .1s ease-in-out
}
.maplibregl-canvas-container.maplibregl-touch-pan-blocker-override.maplibregl-scrollable-page,.maplibregl-canvas-container.maplibregl-touch-pan-blocker-override.maplibregl-scrollable-page .maplibregl-canvas{
    touch-action:pan-x pan-y
}

.map {
    max-width:100%;
    width:100%;
	height:600px;
	margin-right:auto;
    margin-left:auto;

    @media only screen and (max-width: 768px) {
		height:570px !important;
	}
}

.hide {
    @media only screen and (max-width: 768px) {
        display:none;
    }
}

.geocoder {
    top: 8px;
    position: absolute;
    left: 8px;

    @media only screen and (max-width: 768px) {
        display:none;
    }
}
.maplibregl-ctrl-geocoder {
width: 100% !important;
font-size: 15px;
line-height: 20px;
max-width: 100% !important;
}

.maplibregl-ctrl-geocoder ul {
    background-color: #fff;
    border-radius: 0 0 3px 3px;
    left: 0;
    list-style: none;
    margin: 0;
    padding: 0;
    position: absolute;
    width: 100%;
    top: 100%;
    z-index: 1000;
    overflow: hidden;
    font-size: 12px;
}

.maplibregl-ctrl-logo, .maplibregl-ctrl-attrib {
    display:none !important;
  }

.maplibregl-popup{
    z-index: 10 !important; /* or higher if needed */
}

  .legend, .mlegend {
    display:none;
	background-color: rgba(255,255,255,1);
	border-radius: 3px;
	top: 8px;
	box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
	font: 12px/20px graphik-regular;
	padding: 10px;
	position: absolute;
	right: 8px;
    z-index:1 !important;
    color:#333333;
        @media only screen and (max-width: 768px) {

        }
	}


	.legend h4 {
	margin: 0 0 10px;
	}
	 
	.legend div span, .mlegend div span {
    	display: inline-block;
    	height: 8px;
    	margin-right: 0;
    	width: 14px;
    }

    .nodata span  {
        height: 10px !important;
        font-size: 1em;
        color: #888888;
    }

    .nodata  {
        font-size: 1em;
        color: #888888;
    }


	.tooltip {
		width: 400px;
    width: 400px;
		font: 12px/20px graphik-regular;
	}


  .tooltip .maplibregl-popup-content {
  width: 250px !important;
  font-family: graphik-regular, Arial, sans-serif !important;
  font-size: 13px !important;
}

.maplibregl-popup-content table.tableResults {
  width: 100%;
  border-collapse: collapse;
}

.maplibregl-popup-content thead th {
  text-transform: uppercase;
  color: #080808;
  font-size: 10px;
  font-weight: 400;
  border-bottom: 2px solid #eaeaea;
  padding: 3px 4px;
  text-align: left !important;
}

.maplibregl-popup-content tbody td {
  padding: 6px 5px;
  border-bottom: 1px solid #e0e0e0;
  font-size: 12px !important;
  vertical-align: middle;
}

.maplibregl-popup-content .cand {
  vertical-align: middle;
  width: 70%;
  padding-left: 0;
  text-align:left;
  line-height: 14px;
}

.maplibregl-popup-content .votes {
  width: 85px;
  text-align: right;
}

.maplibregl-popup-content .pct {
  width: 65px;
  text-align: right;
  padding-left: 8px;
}

.maplibregl-popup-content .dot {
  width: 9px;
  height: 9px;
  border-radius: 50%;
  display: inline-block;
  background-color: #cfcfcf;
}

.maplibregl-popup-content .dot.DFL {
  background-color: #285e8d;
}

.maplibregl-popup-content .dot.R {
  background-color: #b72e2b;
}

.maplibregl-popup-content .dot.WI {
  background-color: #cfcfcf;
}
.precinct-note {
    font-size: 12px;
    font-style: italic;
    color: #888;
}

.strong {
    opacity:1;
}
.middle {
    opacity:0.6;
}
.weak {
    opacity:0.3;
}

  :global(.monitor_button) {
    display: flex !important;
    align-items: center;
    justify-content: center;
  }

  :global(.cand-list) {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin: 4px 0;
  }

  :global(.cand-row) {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  :global(.cand-name) {
    font-size: 1em;
    line-height: 1.15;
  }

  :global(.cand-ramp) {
    display: flex;
  }

  :global(.cand-ramp span) {
    display: inline-block;
    width: 18px;
    height: 10px;
  }

  .dataline {
    max-width: 100%;
    font-family: graphik-condensed-medium !important;
    font-size: 11px;
    font-weight: 400;
    color: #888 !important;
    margin-bottom: 2px !important;
  }

  .noteline {
    max-width: 100%;
    font-family: graphik-condensed-medium !important;
    font-size: 12px;
    font-weight: 400;
    color: #656565 !important;
    margin-top: 16px !important;
    margin-bottom: 3px;
  }

  .map {
    max-width: 100%;
    width: 100%;
    height: 600px;
    margin-right: auto;
    margin-left: auto;

    @media only screen and (max-width: 768px) {
      height: 570px !important;
    }
  }

  .hide {
    @media only screen and (max-width: 768px) {
      display: none;
    }
  }

  .geocoder {
    top: 8px;
    position: absolute;
    left: 8px;

    @media only screen and (max-width: 768px) {
      display: none;
    }
  }

  :global(.maplibregl-ctrl-geocoder) {
    width: 100% !important;
    font-size: 15px;
    line-height: 20px;
    max-width: 100% !important;
  }

  :global(.maplibregl-ctrl-geocoder ul) {
    background-color: #fff;
    border-radius: 0 0 3px 3px;
    left: 0;
    list-style: none;
    margin: 0;
    padding: 0;
    position: absolute;
    width: 100%;
    top: 100%;
    z-index: 1000;
    overflow: hidden;
    font-size: 12px;
  }

  :global(.maplibregl-ctrl-logo),
  :global(.maplibregl-ctrl-attrib) {
    display: none !important;
  }

  :global(.maplibregl-popup) {
    z-index: 10 !important;
  }

  .legend,
  .mlegend {
    display: none;
    background-color: rgba(255, 255, 255, 1);
    border-radius: 3px;
    top: 8px;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
    font: 12px/20px graphik-regular;
    padding: 10px;
    position: absolute;
    right: 8px;
    z-index: 1 !important;
    color: #333333;
  }

  .legend h4 {
    margin: 0 0 10px;
  }

  .legend div span,
  .mlegend div span {
    display: inline-block;
    height: 8px;
    margin-right: 0;
    width: 14px;
  }

  .nodata span {
    height: 10px !important;
    font-size: 1em;
    color: #888888;
  }

  .nodata {
    font-size: 1em;
    color: #888888;
  }

  .strong {
    opacity: 1;
  }
  .middle {
    opacity: 0.6;
  }
  .weak {
    opacity: 0.3;
  }

  /* ─── Everything below targets MapLibre-injected DOM ─────────────────────── */

  :global(.tooltip) {
    width: 400px;
    font: 12px/20px graphik-regular;
  }

  :global(.tooltip .maplibregl-popup-content) {
    width: 250px !important;
    font-family: graphik-regular, Arial, sans-serif !important;
    font-size: 13px !important;
  }

  :global(.maplibregl-popup-content table.tableResults) {
    width: 100%;
    border-collapse: collapse;
  }

  :global(.maplibregl-popup-content thead th) {
    text-transform: uppercase;
    color: #080808;
    font-size: 10px;
    font-weight: 400;
    border-bottom: 2px solid #eaeaea;
    padding: 3px 4px;
    text-align: left !important;
  }

  :global(.maplibregl-popup-content tbody td) {
    padding: 6px 5px;
    border-bottom: 1px solid #e0e0e0;
    font-size: 12px !important;
    vertical-align: middle;
  }

  :global(.maplibregl-popup-content .cand) {
    vertical-align: middle;
    width: 70%;
    padding-left: 0;
    text-align: left;
    line-height: 14px;
  }

  :global(.maplibregl-popup-content .votes) {
    width: 85px;
    text-align: right;
  }

  :global(.maplibregl-popup-content .pct) {
    width: 65px;
    text-align: right;
    padding-left: 8px;
  }

  :global(.maplibregl-popup-content .dot) {
    width: 9px;
    height: 9px;
    border-radius: 50%;
    display: inline-block;
    background-color: #cfcfcf;
  }

  :global(.maplibregl-popup-content .dot.DFL) {
    background-color: #285e8d;
  }

  :global(.maplibregl-popup-content .dot.R) {
    background-color: #b72e2b;
  }

  :global(.maplibregl-popup-content .dot.WI) {
    background-color: #cfcfcf;
  }

  :global(.precinct-name) {
    display: block;
    font-weight: bold;
  }

  :global(.county-name) {
    display: block;
    color: #888;
    font-size: 11px;
    margin-bottom: 4px;
  }

  :global(.precinct-note) {
    font-size: 12px;
    font-style: italic;
    color: #888;
  }
</style>