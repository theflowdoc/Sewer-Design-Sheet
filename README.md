import streamlit as st
import pandas as pd
import numpy as np
import math
from io import BytesIO
import plotly.express as px

st.set_page_config(
    page_title="Sanitary Sewer Design Tool",
    layout="wide"
)

###############################################################################
# DESIGN PARAMETERS
###############################################################################

AVAILABLE_PIPES = [
    200,
    250,
    300,
    375,
    450,
    525,
    600,
    750,
    900,
    1050,
    1200
]

###############################################################################
# CALCULATIONS
###############################################################################

def harmon_factor(population):

    P = max(population / 1000, 0.001)

    M = 1 + 14 / (4 + math.sqrt(P))

    return max(2, min(4, M))


def peak_domestic_flow(
    population,
    per_capita_flow,
    peaking_factor
):

    return (
        population
        * per_capita_flow
        * peaking_factor
        / 86400
    )


def extraneous_flow(
    cumulative_area,
    infiltration_rate
):

    return cumulative_area * infiltration_rate


def slope(
    us_invert,
    ds_invert,
    length
):

    if length <= 0:
        return 0

    return (
        us_invert
        - ds_invert
    ) / length


def pipe_area(
    diameter_mm
):

    D = diameter_mm / 1000

    return math.pi * D**2 / 4


def manning_capacity(
    diameter_mm,
    slope_value,
    n
):

    if slope_value <= 0:
        return 0

    D = diameter_mm / 1000

    A = math.pi * D**2 / 4

    R = D / 4

    Q = (
        (1 / n)
        * A
        * (R ** (2 / 3))
        * math.sqrt(slope_value)
    )

    return Q * 1000


def velocity(
    diameter_mm,
    capacity_ls
):

    area = pipe_area(diameter_mm)

    if area == 0:
        return 0

    return (capacity_ls / 1000) / area


def capacity_status(utilization):

    if utilization < 0.50:
        return "✅ Good"

    elif utilization < 0.80:
        return "🟡 Monitor"

    return "🔴 Upgrade"


def recommend_pipe(
    design_flow,
    slope_value,
    n
):

    for diameter in AVAILABLE_PIPES:

        capacity = manning_capacity(
            diameter,
            slope_value,
            n
        )

        if capacity >= design_flow:
            return diameter

    return None

###############################################################################
# SIDEBAR
###############################################################################

st.sidebar.title("Design Parameters")

per_capita_flow = st.sidebar.number_input(
    "Per Capita Flow (L/cap/day)",
    value=350.0
)

extraneous_rate = st.sidebar.number_input(
    "Extraneous Flow (L/s/ha)",
    value=0.28
)

population_density = st.sidebar.number_input(
    "Default Population Density (people/ha)",
    value=124.0
)

manning_n = st.sidebar.number_input(
    "Manning n",
    value=0.013,
    format="%.3f"
)

###############################################################################
# TITLE
###############################################################################

st.title("🚧 Sanitary Sewer Design Tool")

st.markdown(
    """
    Upload a sewer GIS export or design sheet CSV.

    Required columns:

    - Street
    - FromMH
    - ToMH
    - Area_ha
    - Length_m
    - Diameter_mm
    - UpstreamInvert
    - DownstreamInvert

    Optional:

    - PopDensity
    """
)

###############################################################################
# FILE UPLOAD
###############################################################################

uploaded_file = st.file_uploader(
    "Upload CSV",
    type=["csv"]
)

###############################################################################
# SAMPLE TABLE
###############################################################################

with st.expander("Example CSV Format"):

    example = pd.DataFrame({
        "Street":["Street A"],
        "FromMH":["MH-1"],
        "ToMH":["MH-2"],
        "Area_ha":[0.65],
        "Length_m":[75],
        "Diameter_mm":[200],
        "UpstreamInvert":[103.55],
        "DownstreamInvert":[103.20],
        "PopDensity":[124]
    })

    st.dataframe(example)

###############################################################################
# PROCESS
###############################################################################

if uploaded_file:

    df = pd.read_csv(uploaded_file)

    cumulative_population = 0
    cumulative_area = 0

    results = []

    for idx, row in df.iterrows():

        area = row["Area_ha"]

        density = row.get(
            "PopDensity",
            population_density
        )

        population = area * density

        cumulative_population += population
        cumulative_area += area

        peaking = harmon_factor(
            cumulative_population
        )

        q_peak = peak_domestic_flow(
            cumulative_population,
            per_capita_flow,
            peaking
        )

        q_infiltration = extraneous_flow(
            cumulative_area,
            extraneous_rate
        )

        q_total = (
            q_peak +
            q_infiltration
        )

        pipe_slope = slope(
            row["UpstreamInvert"],
            row["DownstreamInvert"],
            row["Length_m"]
        )

        capacity = manning_capacity(
            row["Diameter_mm"],
            pipe_slope,
            manning_n
        )

        vel = velocity(
            row["Diameter_mm"],
            capacity
        )

        utilization = (
            q_total / capacity
            if capacity > 0
            else 999
        )

        recommended = recommend_pipe(
            q_total,
            pipe_slope,
            manning_n
        )

        results.append({

            "Street":
                row["Street"],

            "From":
                row["FromMH"],

            "To":
                row["ToMH"],

            "Area (ha)":
                round(area,2),

            "Population":
                round(cumulative_population),

            "Peaking Factor":
                round(peaking,2),

            "Q Peak (L/s)":
                round(q_peak,2),

            "Q Infiltration (L/s)":
                round(q_infiltration,2),

            "Q Design (L/s)":
                round(q_total,2),

            "Slope (%)":
                round(pipe_slope*100,3),

            "Diameter (mm)":
                row["Diameter_mm"],

            "Capacity (L/s)":
                round(capacity,2),

            "Velocity (m/s)":
                round(vel,2),

            "Utilization":
                round(utilization,2),

            "Recommended Pipe":
                recommended,

            "Status":
                capacity_status(utilization)
        })

    results_df = pd.DataFrame(results)

    ###########################################################################
    # RESULTS
    ###########################################################################

    st.subheader("Results")

    st.dataframe(
        results_df,
        use_container_width=True
    )

    ###########################################################################
    # SUMMARY
    ###########################################################################

    st.subheader("System Summary")

    c1, c2, c3, c4 = st.columns(4)

    c1.metric(
        "Total Population",
        int(results_df["Population"].max())
    )

    c2.metric(
        "Max Design Flow",
        round(
            results_df["Q Design (L/s)"].max(),
            2
        )
    )

    c3.metric(
        "Average Utilization",
        round(
            results_df["Utilization"].mean(),
            2
        )
    )

    c4.metric(
        "Max Capacity",
        round(
            results_df["Capacity (L/s)"].max(),
            2
        )
    )

    ###########################################################################
    # CHARTS
    ###########################################################################

    st.subheader("Capacity Utilization")

    fig1 = px.bar(
        results_df,
        x="From",
        y="Utilization",
        color="Utilization"
    )

    st.plotly_chart(
        fig1,
        use_container_width=True
    )

    st.subheader("Design Flow")

    fig2 = px.line(
        results_df,
        x="From",
        y="Q Design (L/s)",
        markers=True
    )

    st.plotly_chart(
        fig2,
        use_container_width=True
    )

    ###########################################################################
    # EXPORT
    ###########################################################################

    output = BytesIO()

    with pd.ExcelWriter(
        output,
        engine="openpyxl"
    ) as writer:

        results_df.to_excel(
            writer,
            index=False,
            sheet_name="Results"
        )

    st.download_button(
        label="📥 Download Excel Results",
        data=output.getvalue(),
        file_name="Sewer_Design_Results.xlsx",
        mime=(
            "application/vnd.openxmlformats-"
            "officedocument.spreadsheetml.sheet"
        )
    )
