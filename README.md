import streamlit as st
import cohere
import requests
import datetime
import time
import threading
import os
from collections import deque

# 🔐 API Keys
COHERE_API_KEY = "kf2yELmUe4p3CksYfbBbvgh2Hk9xgCgIr7BAjlds"
WEATHER_API_KEY = "bd5e378503939ddaee76f12ad7a97608"
MAPBOX_API_KEY = "pk.eyeyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw"  # For route planning

co = cohere.Client(COHERE_API_KEY)

# --- Custom CSS for enhanced styling ---
st.markdown(
    """
    <style>
        /* (Keep your existing CSS styles here) */
        .stop-card {
            background-color: #f8f9fa;
            border-radius: 8px;
            padding: 15px;
            margin-bottom: 10px;
            border-left: 4px solid #007bff;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .current-stop {
            border-left: 4px solid #28a745;
            background-color: #e8f4f8;
        }
        .progress-container {
            margin: 20px 0;
            background-color: #e9ecef;
            border-radius: 5px;
            height: 10px;
        }
        .progress-bar {
            height: 100%;
            border-radius: 5px;
            background-color: #007bff;
            width: 0%;
            transition: width 0.5s ease;
        }
    </style>
    """,
    unsafe_allow_html=True,
)

# 🌦️ Fetch real-time weather
def get_weather(city):
    url = f"https://api.openweathermap.org/data/2.5/weather?q={city}&appid={WEATHER_API_KEY}&units=metric"
    try:
        response = requests.get(url)
        data = response.json()
        if response.status_code == 200:
            description = data['weather'][0]['description']
            temp = data['main']['temp']
            return f"🌤️ Weather in {city}: {description}, {temp}°C"
        else:
            return "Weather info not available."
    except:
        return "Failed to fetch weather info."

# 🗺️ Get route with intermediate stops (simulated)
def get_route_stops(source, destination, travel_type):
    # In a real app, you would use Mapbox Directions API or similar
    # This is a simulation with common stops based on travel type
    
    stops = []
    
    if travel_type.lower() == "car":
        stops = [
            {"name": f"{source} City Limits", "distance": 0, "duration": 0},
            {"name": "Highway Junction", "distance": 25, "duration": 30},
            {"name": "Rest Area", "distance": 60, "duration": 75},
            {"name": "Major Intersection", "distance": 100, "duration": 120},
            {"name": f"{destination} Outskirts", "distance": 150, "duration": 180},
        ]
    elif travel_type.lower() == "train":
        stops = [
            {"name": f"{source} Station", "distance": 0, "duration": 0},
            {"name": "First Stop", "distance": 50, "duration": 45},
            {"name": "Second Stop", "distance": 100, "duration": 90},
            {"name": "Transfer Station", "distance": 150, "duration": 135},
            {"name": f"{destination} Station", "distance": 200, "duration": 180},
        ]
    elif travel_type.lower() == "bus":
        stops = [
            {"name": f"{source} Bus Terminal", "distance": 0, "duration": 0},
            {"name": "Downtown Stop", "distance": 20, "duration": 30},
            {"name": "Shopping District", "distance": 40, "duration": 60},
            {"name": "Suburban Transfer", "distance": 60, "duration": 90},
            {"name": f"{destination} Bus Station", "distance": 80, "duration": 120},
        ]
    elif travel_type.lower() == "plane":
        stops = [
            {"name": f"{source} Airport", "distance": 0, "duration": 0},
            {"name": "In-Flight", "distance": 500, "duration": 60},
            {"name": f"{destination} Airport", "distance": 1000, "duration": 120},
        ]
    else:  # bike
        stops = [
            {"name": f"{source} Starting Point", "distance": 0, "duration": 0},
            {"name": "Park Entrance", "distance": 10, "duration": 30},
            {"name": "Scenic Viewpoint", "distance": 20, "duration": 60},
            {"name": "Rest Stop", "distance": 30, "duration": 90},
            {"name": f"{destination} Arrival", "distance": 40, "duration": 120},
        ]
    
    # Add weather for each stop
    for stop in stops:
        stop['weather'] = get_weather(stop['name'].split(' ')[0]) if not any(x in stop['name'].lower() for x in ['junction', 'area', 'intersection', 'outskirts', 'in-flight']) else "Weather not available"
    
    return stops

# 🤖 Generate AI tour details with Cohere
def generate_tour_details(source, destination, travel_type, stops):
    prompt = f"""
    You are an AI assistant enhancing tourist experiences. Provide detailed information based on the following:

    Source: {source}
    Destination: {destination}
    Travel Type: {travel_type}
    Route Stops: {[stop['name'] for stop in stops]}
    
    Include:
    1. Overview of the journey
    2. Key attractions at each stop
    3. Travel tips for {travel_type} travel
    4. Estimated time between stops
    5. Any special considerations
    """

    response = co.generate(
        model='command-r-plus',
        prompt=prompt,
        max_tokens=1000,
        temperature=0.8
    )
    return response.generations[0].text.strip()

# ⏰ Reminder thread
def reminder_popup(message, delay):
    time.sleep(delay)
    st.toast(message)

# 🚀 Streamlit App
st.title("🧳 AI Tourist Experience App")

with st.form("tour_form"):
    col1, col2 = st.columns(2)
    with col1:
        source = st.text_input("Source", placeholder="New York")
    with col2:
        destination = st.text_input("Destination", placeholder="Boston")
    
    travel_type = st.selectbox("Travel Type", ["Car", "Train", "Bus", "Plane", "Bike"])
    arrival_time = st.text_input("Estimated Arrival Time (YYYY-MM-DD HH:MM)", placeholder="2024-06-15 14:30")
    submitted = st.form_submit_button("Generate Itinerary")

if submitted:
    # Get route with stops
    route_stops = get_route_stops(source, destination, travel_type)
    
    with st.spinner("Generating itinerary..."):
        details = generate_tour_details(source, destination, travel_type, route_stops)
        destination_weather = get_weather(destination)

    st.markdown("### 📍 Tour Details")
    st.markdown(details.replace("\n", "  \n"))

    st.markdown("### 🌦️ Weather at Destination")
    st.info(destination_weather)

    st.markdown("### 🗺️ Route Progress")
    
    # Create a progress bar
    progress = st.empty()
    progress_bar = progress.progress(0)
    
    # Display stops
    stops_container = st.container()
    
    # Simulate real-time progress (in a real app, this would come from GPS)
    for i, stop in enumerate(route_stops):
        # Update progress
        progress_value = (i / (len(route_stops) - 1)) * 100 if len(route_stops) > 1 else 100
        progress_bar.progress(int(progress_value))
        
        # Display stops with current stop highlighted
        with stops_container:
            col1, col2 = st.columns([3, 1])
            with col1:
                stop_class = "stop-card current-stop" if i == 0 else "stop-card"
                st.markdown(f"""
                <div class="{stop_class}">
                    <h4>{stop['name']}</h4>
                    <p>Distance: {stop['distance']} km | Time: {stop['duration']} min</p>
                    <p><small>{stop['weather']}</small></p>
                </div>
                """, unsafe_allow_html=True)
            
            with col2:
                if i < len(route_stops) - 1:
                    next_stop = route_stops[i+1]
                    st.metric("Next Stop", next_stop['name'], 
                             f"{next_stop['distance'] - stop['distance']} km")
        
        # Simulate time between stops (remove in production)
        if i < len(route_stops) - 1:
            time.sleep(2)  # Simulate time between stops

    st.markdown("### 🗺️ Google Maps")
    maps_url = f"https://www.google.com/maps/dir/{source.replace(' ', '+')}/{destination.replace(' ', '+')}"
    st.markdown(f"[View Route from {source} to {destination} on Google Maps]({maps_url})")

    # 🕐 Handle 10-min arrival reminder
    if arrival_time:
        try:
            arrival_datetime = datetime.datetime.strptime(arrival_time, "%Y-%m-%d %H:%M")
            reminder_time = arrival_datetime - datetime.timedelta(minutes=10)
            now = datetime.datetime.now()
            if reminder_time > now:
                wait_seconds = (reminder_time - now).total_seconds()
                st.success(f"✅ Reminder set for {reminder_time.strftime('%H:%M')}. You will be notified.")
                threading.Thread(
                    target=reminder_popup,
                    args=("🔔 Your destination is arriving in 10 minutes. Please prepare to exit.", wait_seconds),
                    daemon=True
                ).start()
            else:
                st.warning("⚠️ Reminder time has already passed.")
        except ValueError:
            st.error("❌ Invalid time format. Please use YYYY-MM-DD HH:MM.")
