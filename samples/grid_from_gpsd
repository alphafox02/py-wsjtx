import threading
from datetime import datetime
import sys
import os

sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), '..')))

import pywsjtx
import pywsjtx.extra.simple_server
from pywsjtx.extra.latlong_to_grid_square import LatLongToGridSquare

import gps

IP_ADDRESS = '127.0.0.1'
PORT = 2237


class NMEALocation:
    def __init__(self, grid_changed_callback=None):
        self.valid = False
        self.grid = ""
        self.last_fix_at = None
        self.grid_changed_callback = grid_changed_callback
        self.session = gps.gps(mode=gps.WATCH_ENABLE | gps.WATCH_NEWSTYLE)

    def handle_gpsd(self):
        while True:
            try:
                report = self.session.next()
                if report["class"] == "TPV":
                    latitude = report.get("lat")
                    longitude = report.get("lon")
                    if latitude is not None and longitude is not None:
                        nmea_sentence = self.generate_nmea_sentence(latitude, longitude)
                        print("Received NMEA sentence: {}".format(nmea_sentence))
                        if nmea_sentence.startswith('$GPGLL'):
                            grid = LatLongToGridSquare().to_grid(latitude, longitude)
                            if grid:
                                self.grid = grid
                                self.valid = True
                                self.last_fix_at = datetime.utcnow()
                                if self.grid_changed_callback:
                                    c_thr = threading.Thread(target=self.grid_changed_callback, args=(self.grid,),
                                                             kwargs={})
                                    c_thr.start()
            except Exception as e:
                print("Error: {}".format(e))

    def generate_nmea_sentence(self, latitude, longitude):
        nmea_sentence = "$GPGLL,{:.6f},N,{:.6f},W".format(latitude, longitude)
        return nmea_sentence


class WSJTXManager:
    def __init__(self):
        self.wsjtx_id = None
        self.nmea_location = None
        self.gps_grid = ""
        self.nmea_location = NMEALocation()

    def start(self):
        s = pywsjtx.extra.simple_server.SimpleServer(IP_ADDRESS, PORT)

        while True:
            (pkt, addr_port) = s.rx_packet()
            if pkt is not None:
                the_packet = pywsjtx.WSJTXPacketClassFactory.from_udp_packet(addr_port, pkt)
                if self.wsjtx_id is None and isinstance(the_packet, pywsjtx.HeartBeatPacket):
                    print("wsjtx detected, id is {}".format(the_packet.wsjtx_id))
                    print("starting gps monitoring")
                    self.wsjtx_id = the_packet.wsjtx_id
                    self.nmea_location = NMEALocation(self.callback)
                    self.start_gps_monitoring()

                if isinstance(the_packet, pywsjtx.StatusPacket):
                    if self.gps_grid != "" and the_packet.de_grid != self.gps_grid:
                        print("Sending Grid Change to wsjtx-x, old grid:{} new grid: {}".format(the_packet.de_grid,
                                                                                                self.gps_grid))
                        grid_change_packet = pywsjtx.LocationChangePacket.Builder(self.wsjtx_id,
                                                                                  "GRID:" + self.gps_grid)
                        s.send_packet(addr_port, grid_change_packet)

                print(the_packet)

    def start_gps_monitoring(self):
        gps_thread = threading.Thread(target=self.nmea_location.handle_gpsd)
        gps_thread.daemon = True
        gps_thread.start()

    def callback(self, new_grid):
        if new_grid != self.gps_grid:
            print("New Grid! {}".format(new_grid))
            self.gps_grid = new_grid


wsjtx_manager = WSJTXManager()
wsjtx_manager.start()
