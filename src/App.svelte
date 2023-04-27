<script>
  // CORE IMPORTS
  import { setContext, onMount } from 'svelte'
  import { getMotion } from './utils.js'
  import { themes } from './config.js'
  import ONSHeader from './layout/ONSHeader.svelte'
  import ONSFooter from './layout/ONSFooter.svelte'
  import Header from './layout/Header.svelte'
  import Section from './layout/Section.svelte'
  import Media from './layout/Media.svelte'
  import Scroller from './layout/Scroller.svelte'
  import Filler from './layout/Filler.svelte'
  import Divider from './layout/Divider.svelte'
  import Toggle from './ui/Toggle.svelte'
  import Arrow from './ui/Arrow.svelte'
  import Em from './ui/Em.svelte'

  // DEMO-SPECIFIC IMPORTS
  import bbox from '@turf/bbox'
  import { getData, setColors, getTopo, getBreaks, getColor } from './utils.js'
  import { colors, units } from './config.js'
  import { ScatterChart, LineChart, BarChart } from '@onsvisual/svelte-charts'
  import { Map, MapSource, MapLayer, MapTooltip } from '@onsvisual/svelte-maps'



  // CORE CONFIG (COLOUR THEMES)
  // Set theme globally (options are 'light', 'dark' or 'lightblue')
  let theme = 'light'
  setContext('theme', theme)
  setColors(themes, theme)

  // CONFIG FOR SCROLLER COMPONENTS
  // Config
  const threshold = 0.65
  // State
  let animation = getMotion() // Set animation preference depending on browser preference
  let id = {} // Object to hold visible section IDs of Scroller components
  let idPrev = {} // Object to keep track of previous IDs, to compare for changes
  onMount(() => {
    idPrev = { ...id }

  })

  // DEMO-SPECIFIC CONFIG
  // Constants
  const datasets = ['region', 'district']
  const topojson = './data/lga.json'
  const mapstyle = 'https://bothness.github.io/ons-basemaps/data/style-omt.json'
  const mapbounds = {
    uk: [
      [12, 14],
      [7, 5],
    ],
  }

  // Data
  let data = { district: {}, region: {} }
  let metadata = { district: {}, region: {} }
  let geojson

  // Element bindings
  let map = null // Bound to mapbox 'map' instance once initialised

  // State
  let hovered // Hovered district (chart or map)
  let selected // Selected district (chart or map)
  $: region =
    selected && metadata.district.lookup
      ? metadata.district.lookup[selected].parent
      : null // Gets region code for 'selected'
  $: chartHighlighted =
    metadata.district.array && region
      ? metadata.district.array
          .filter((d) => d.parent == region)
          .map((d) => d.code)
      : [] // Array of district codes in 'region'
  let mapHighlighted = [] // Highlighted district (map only)
  let xKey = 'area' // xKey for scatter chart
  let yKey = null // yKey for scatter chart
  let zKey = null // zKey (color) for scatter chart
  let rKey = null // rKey (radius) for scatter chart
  let mapKey = 'density' // Key for data to be displayed on map
  let explore = false // Allows chart/map interactivity to be toggled on/off

  // FUNCTIONS (INCL. SCROLLER ACTIONS)

  // Functions for chart and map on:select and on:hover events
  function doSelect(e) {
    console.log(e)
    selected = e.detail.id
    if (e.detail.feature) fitById(selected) // Fit map if select event comes from map
  }
  function doHover(e) {
    hovered = e.detail.id
  }

  // Functions for map component
  function fitBounds(bounds) {
    if (map) {
      map.fitBounds(bounds, { animate: animation, padding: 30 })
    }
  }
  function fitById(id) {
    if (geojson && id) {
      let feature = geojson.features.find((d) => d.properties.AREACD == id)
      let bounds = bbox(feature.geometry)
      fitBounds(bounds)
    }
  }

  // Actions for Scroller components
  const actions = {
    map: {
      // Actions for <Scroller/> with id="map"
      map01: () => {
        // Action for <section/> with data-id="map01"
        fitBounds(mapbounds.uk)
        mapKey = 'density'
        mapHighlighted = []
        explore = false
      },
      map02: () => {
        fitBounds(mapbounds.uk)
        mapKey = 'age_med'
        mapHighlighted = []
        explore = false
      },
      map03: () => {
        let hl = [...data.district.indicators].sort(
          (a, b) => b.age_med - a.age_med,
        )[0]
        fitById(hl.code)
        mapKey = 'age_med'
        mapHighlighted = [hl.code]
        explore = false
      },
      map04: () => {
        fitBounds(mapbounds.uk)
        mapKey = 'age_med'
        mapHighlighted = []
        explore = true
      },
    },
    chart: {
      chart01: () => {
        xKey = 'area'
        yKey = null
        zKey = null
        rKey = null
        explore = true
      },
      chart02: () => {
        xKey = 'area'
        yKey = null
        zKey = null
        rKey = 'pop'
        explore = true
      },
      chart03: () => {
        xKey = 'area'
        yKey = 'density'
        zKey = null
        rKey = 'pop'
        explore = true
      },
      chart04: () => {
        xKey = 'area'
        yKey = 'density'
        zKey = 'parent_name'
        rKey = 'pop'
        explore = false
      },
      chart05: () => {
        xKey = 'area'
        yKey = 'density'
        zKey = null
        rKey = 'pop'
        explore = true
      },
    },
  }

const date = new Date();
const months = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
const monthName = months[date.getMonth()];
const currentDate = `${date.getDate()} ${monthName} ${date.getFullYear()}`;


  // Code to run Scroller actions when new caption IDs come into view
  function runActions(codes = []) {
    codes.forEach((code) => {
      if (id[code] != idPrev[code]) {
        if (actions[code][id[code]]) {
          actions[code][id[code]]()
        }
        idPrev[code] = id[code]
      }
    })
  }
  $: id && runActions(Object.keys(actions)) // Run above code when 'id' object changes

  // INITIALISATION CODE
  datasets.forEach((geo) => {
    getData(`./data/data_${geo}.csv`).then((arr) => {
      let meta = arr.map((d) => ({
        code: d.code,
        name: d.name,
        parent: d.parent ? d.parent : null,
      }))
      let lookup = {}
      meta.forEach((d) => {
        lookup[d.code] = d
      })
      metadata[geo].array = meta
      metadata[geo].lookup = lookup

      let indicators = arr.map((d, i) => ({
        ...meta[i],
        area: d.area,
        pop: d['2020'],
        density: d.density,
        age_med: d.age_med,
      }))

      if (geo == 'district') {
        ;['density', 'age_med'].forEach((key) => {
          let values = indicators.map((d) => d[key]).sort((a, b) => a - b)
          let breaks = getBreaks(values)
          indicators.forEach(
            (d, i) =>
              (indicators[i][key + '_color'] = getColor(
                d[key],
                breaks,
                colors.seq,
              )),
          )
        })
      }
      data[geo].indicators = indicators

      let years = [
        2001,
        2002,
        2003,
        2004,
        2005,
        2006,
        2007,
        2008,
        2009,
        2010,
        2011,
        2012,
        2013,
        2014,
        2015,
        2016,
        2017,
        2018,
        2019,
        2020,
      ]

      let timeseries = []
      arr.forEach((d) => {
        years.forEach((year) => {
          timeseries.push({
            code: d.code,
            name: d.name,
            value: d[year],
            year,
          })
        })
      })
      data[geo].timeseries = timeseries
    })
  })

  getTopo(topojson, 'geog').then((geo) => {
    geo.features.sort((a, b) =>
      a.properties.AREANM.localeCompare(b.properties.AREANM),
    )
    geojson = geo
  })
  


</script>

<style>
  /* Styles specific to elements within the demo */
  :global(svelte-scroller-foreground) {
    pointer-events: none !important;
  }
  :global(svelte-scroller-foreground section div) {
    pointer-events: all !important;
  }
  select {
    max-width: 350px;
  }
  .chart {
    margin-top: 45px;
    width: calc(100% - 5px);
  }
  .chart-full {
    margin: 0 20px;
  }
  .chart-sml {
    font-size: 0.85em;
  }
  /* The properties below make the media DIVs grey, for visual purposes in demo */
  .media {
    background-color: #f0f0f0;
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    -webkit-box-orient: vertical;
    -webkit-box-direction: normal;
    -ms-flex-flow: column;
    flex-flow: column;
    -webkit-box-pack: center;
    -ms-flex-pack: center;
    justify-content: center;
    text-align: center;
    color: #aaa;
  }
</style>

<ONSHeader filled={true} center={false} />
<Header
  bgcolor="#0FFFF"
  bgfixed={true}
  theme="dark"
  center={false}
  short={true}>
  
  <h1 style="margin-top: 5px; color: green">E-NAIRA: The Journey</h1>
  <p class="text-big" style="margin-top: 5px; color: green">
    This is a short text description of e-naira usage across Nigeria.
  </p>
  <p style="margin-top: 10px; color: green ">{currentDate}</p>
  <!-- <p>
    <Toggle
      label="Animation {animation ? 'on' : 'off'}"
      mono={true}
      bind:checked={animation} />
  </p>
  <div style="margin-top: 0px;">
    <Arrow color="white" {animation}>Scroll to begin</Arrow>
  </div> -->
</Header> 


<!-- <Filler theme="lightblue" short={true} wide={true} center={false}>
  
  <p class="text-big">This is a large, left-aligned text caption</p>

</Filler> -->

<Section>
  <h2>E-Naira Policy</h2>
  <p>
    decrease in the use of physical cash, with most
transactions being settled electronically through
methods such as online banking, internet
banking, Point of Sale (PoS), and Automated
Teller Machines (ATMs). This has boosted ecommerce activities both domestically and
cross-border, with cashless transactions
recorded in Nigeria between January and April
2021 totaling N81.54tn. According to data from
the Nigeria Inter-Bank Settlement System
(NIBSS, 2022), this amount increased by 44
percent year-over-year to reach N117.33tn in
2022.
  </p>
  <p>
    What are the direct benefits of the eNaira?
  </p>
  <blockquote class="text-indent">
    "The direct benefits of issuing eNaira vary depending on the specific context, but some
    potential benefits include:
    <br/>
    Improved financial inclusion:
    By providing a digital alternative to cash, eNaira
can make it easier for people who are unbanked
or underbanked to access financial services.
  "
  </blockquote>
</Section>

<Divider />

<Section>
  <h2>Distribution and circulation of e-naira</h2>
  <p>
    Below is a distribution chart of e-naira. It is display the distribution within the states of Nigeria.
  </p>
</Section>

{#if data.region.indicators}
  <Media col="wide" caption="Source: CBN e-naira states estimates.">
    <div class="chart-sml">
      <BarChart
        data={[...data.region.indicators].sort((a, b) => a.pop - b.pop)}
        xKey="pop"
        yKey="name"
        snapTicks={false}
        xFormatTick={(d) => d / 1e6}
        xSuffix="m"
        color="green"
        height={350}
        padding={{ top: 20, bottom: 15, left: 140, right: 0 }}
        area={false}
        title="E-nairra circulation by region/nation, 2022" />
       
    </div>
  </Media>
{/if}

<Divider />

<Section>
  <h2>Transaction charts of e-naira of states </h2>
  <p>
    eNaira can provide more accurate and granular 
data on economic activity and monetary policy 
transmission mechanisms.
It's important to note that the Bank is aware of 
the potential risks and challenges associated 
with issuing eNaira, such as the potential for 
increased financial instability, privacy concerns, 
and the need for significant investments in new 
technology and infrastructure. Adequate 
measures are in place to ensure that this risk is
mitigated to the barest minimum.

  </p>
</Section>

{#if data.region.timeseries && data.region.indicators}
  <Media
    col="wide"
    grid="narrow"
    gap={20}
    caption="Source: CBN e-naira circulation estimates.">
    {#each [...data.region.indicators].sort((a, b) => b.pop - a.pop) as region}
      <div class="chart-sml">
        <LineChart
          data={data.region.timeseries}
          xKey="year"
          yKey="value"
          zKey="code"
          color="green"
          lineWidth={1}
          xTicks={2}
          snapTicks={false}
          yFormatTick={(d) => d / 1e6}
          ySuffix="m"
          height={200}
          padding={{ top: 0, bottom: 20, left: 30, right: 15 }}
          selected={region.code}
          area={false}
          title={region.name} />
      </div>
    {/each}
  </Media>
{/if}

<Divider />

<Section>
  <h2>The usage of e-naira in LGAs of Nigeria</h2>
  <p>
    eNaira is a digital version of Nigeria’s
fiat currency, issued and backed by the 
country's central bank. It is designed to coexist 
with physical Naira (cash) and bank deposits 
and can be used for making digital payments, 
just like other digital currencies.
  </p>
  <p>
    Here are some preventive measures that are
taken by CBN and other financial institutions to 
prevent fraud:
  </p>
</Section>

<Scroller {threshold} bind:id={id['chart']} splitscreen={true}>
  <div slot="background">
    <figure>
      <div class="col-wide height-full">
        {#if data.district.indicators && metadata.region.lookup}
          <div class="chart">
            <ScatterChart
              height="calc(100vh - 150px)"
              data={data.district.indicators.map((d) => ({
                ...d,
                parent_name: metadata.region.lookup[d.parent].name,
              }))}
              colors={explore ? ['green'] : colors.cat}
              {xKey}
              {yKey}
              {zKey}
              {rKey}
              idKey="code"
              labelKey="name"
              r={[3, 10]}
              xScale="log"
              xTicks={[10, 100, 1000, 10000]}
              xFormatTick={(d) => d.toLocaleString()}
              xSuffix=" sq.km"
              yFormatTick={(d) => d.toLocaleString()}
              legend={zKey != null}
              labels
              select={explore}
              selected={explore ? selected : null}
              on:select={doSelect}
              hover
              {hovered}
              on:hover={doHover}
              highlighted={explore ? chartHighlighted : []}
              colorSelect="#206095"
              colorHighlight="#999"
              overlayFill
              {animation} />
          </div>
        {/if}
      </div>
    </figure>
  </div>

  <div slot="foreground">
    <section data-id="chart01">
      <div class="col-medium">
        <p>
          This chart shows the
          <strong>e-naira transaction in all LGAs in Nigeia.</strong>
           Each circle represents one district of local authority district in the Nigeria.
        </p>
      </div>
    </section>
    <section data-id="chart02">
      <div class="col-medium">
        <p>
          The following chart shows the growth rate 
          <strong>of e-naira transaction </strong>
          in all LGAs.
        </p>
      </div>
    </section>
    <section data-id="chart03">
      <div class="col-medium">
        <p>
           This shows the usage density 
          <strong>of e-naira </strong>
          per people of the LGAs.
        </p>
      </div>
    </section>
    <section data-id="chart04">
      <div class="col-medium">
        <p>
          The colour of each circle (LGAs) shows the
          <strong>the gap needed in transcation of e-naira</strong>
          that the LGAs is within.
        </p>
      </div>
    </section>
    <section data-id="chart05">
      <div class="col-medium">
        <h3>Select LGA</h3>
        <p>
          The chart will also highlight the other LGAs with most usage e-naira in the country.
        </p>
        {#if geojson}
          <p>
            <!-- svelte-ignore a11y-no-onchange -->
            <select bind:value={selected}>
              <option value={null}>Select one</option>
              {#each geojson.features as place}
                <option value={place.properties.AREACD}>
                  {place.properties.AREANM}
                </option>
              {/each}
            </select>
          </p>
        {/if}
      </div>
    </section>
  </div>
</Scroller>

<Divider />

<Section>
  <h2>This standard deviation of e-naira </h2>
  <p>
    The overriding objective for introducing the 
eNaira is that households and businesses can 
make fast, efficient, and reliable payments, and 
benefit from a resilient, inclusive, innovative, and 
inexpensive payment system.
  </p>
  <p />
</Section>

<Media
  col="full"
  height={600}
  caption="Line distribution of e-naira ">
  <div class="chart-full">
    {#if data.district.timeseries}
      <LineChart
        data={data.district.timeseries}
        padding={{ left: 50, right: 150, top: 0, bottom: 0 }}
        height="500px"
        xKey="year"
        yKey="value"
        zKey="code"
        color="green"
        lineWidth={1}
        yFormatTick={(d) => (d / 1e6).toFixed(1)}
        ySuffix="m"
        select
        {selected}
        on:select={doSelect}
        hover
        {hovered}
        on:hover={doHover}
        highlighted={chartHighlighted}
        colorSelect="#206095"
        colorHighlight="#999"
        area={false}
        title="2years transction by district, 2020 to 2022"
        labels
        labelKey="name" />
    {/if}
  </div>
</Media>

<Divider />

<Section>
  <h2>The shows the area and circulation of e-naira</h2>
  <p class="mb">
    The below map shows the details area of the most and least usage and adoption of e-naira
  </p>
</Section>

{#if geojson && data.district.indicators}
  <Scroller {threshold} bind:id={id['map']}>
    <div slot="background">
      <figure>
        <div class="col-full height-full">
          <Map
            style={mapstyle}
            bind:map
            interactive={false}
            location={{ bounds: mapbounds.uk }}>
            <MapSource
              id="lad"
              type="geojson"
              data={geojson}
              promoteId="AREACD"
              maxzoom={13}>
              <MapLayer
                id="lad-fill"
                idKey="code"
                colorKey={mapKey + '_color'}
                data={data.district.indicators}
                type="fill"
                select
                {selected}
                on:select={doSelect}
                clickIgnore={!explore}
                hover
                {hovered}
                on:hover={doHover}
                highlight
                highlighted={mapHighlighted}
                paint={{ 'fill-color': ['case', ['!=', ['feature-state', 'color'], null], ['feature-state', 'color'], 'rgba(255, 255, 255, 0)'], 'fill-opacity': 0.7 }}>
                <MapTooltip
                  content={hovered ? `${metadata.district.lookup[hovered].name}<br/><strong>${data.district.indicators
                        .find((d) => d.code == hovered)
                        [
                          mapKey
                        ].toLocaleString()} ${units[mapKey]}</strong>` : ''} />
              </MapLayer>
              <MapLayer
                id="lad-line"
                type="line"
                paint={{ 'line-color': ['case', ['==', ['feature-state', 'hovered'], true], 'orange', ['==', ['feature-state', 'selected'], true], 'black', ['==', ['feature-state', 'highlighted'], true], 'black', 'rgba(255,255,255,0)'], 'line-width': 2 }} />
            </MapSource>
          </Map>
        </div>
      </figure>
    </div>

    <div slot="foreground">
      <section data-id="map01">
        <div class="col-medium">
          <p>
            This map display
            <strong>e-naira circulation density</strong>
            by LGAs. LGAs are coloured from
            <Em color={colors.seq[0]}>least dense</Em>
            to
            <Em color={colors.seq[4]}>most dense</Em>
            . You can hover to see the LGA name and circulation density.
          </p>
        </div>
      </section>
      <section data-id="map02">
        <div class="col-medium">
          <p>
            The map then display
            <strong> the average usage of e-naira</strong>
            , from
            <Em color={colors.seq[0]}>least usage</Em>
            to
            <Em color={colors.seq[4]}>most usage</Em>
            .
          </p>
        </div>
      </section>
      <section data-id="map03">
        <div class="col-medium">
          <!-- This gets the data object for the district with the oldest median age -->
          {#each [[...data.district.indicators].sort((a, b) => b.age_med - a.age_med)[0]] as district}
            <p>
              The map is now zoomed on
              <Em color={district.age_med_color}>{district.name}</Em>
              , the district with the most  usage of e-naira, with {district.age_med}
               in 2years.
            </p>
          {/each}
        </div>
      </section>
      <section data-id="map04">
        <div class="col-medium">
          <h3>Select a LGA</h3>
          <p>
            Use the selection box below or click on the map to select and zoom
            to a LGA.
          </p>
          {#if geojson}
            <p>
              <!-- svelte-ignore a11y-no-onchange -->
              <select bind:value={selected} on:change={() => fitById(selected)}>
                <option value={null}>Select one</option>
                {#each geojson.features as place}
                  <option value={place.properties.AREACD}>
                    {place.properties.AREANM}
                  </option>
                {/each}
              </select>
            </p>
          {/if}
        </div>
      </section>
    </div>
  </Scroller>
{/if}

<Divider />
<!--
<Section>
  <h2>How to use this template</h2>
  <p>
    You can find the source code and documentation on how to use this template
    in
    <a href="https://github.com/ONSvisual/svelte-scrolly/" target="_blank">
      this Github repo
    </a>
    .
  </p>
</Section>
-->
<ONSFooter />
