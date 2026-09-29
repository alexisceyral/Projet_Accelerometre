import time
import signal
import sys
from datetime import datetime

import smbus2
import openpyxl
from luma.core.interface.serial import i2c
from luma.core.render import canvas
from luma.oled.device import ssd1306


class CapteurMPU6050:
    def __init__(self, adresse=0x68, bus_id=1):
        self.bus = smbus2.SMBus(bus_id)
        self.adresse = adresse
        self.bus.write_byte_data(self.adresse, 0x6B, 0)

    def lire_mot(self, registre):
        haut = self.bus.read_byte_data(self.adresse, registre)
        bas = self.bus.read_byte_data(self.adresse, registre + 1)
        valeur = (haut << 8) + bas
        if valeur >= 0x8000:
            valeur -= 0x10000
        return valeur

    def lire_acceleration(self):
        g_x = self.lire_mot(0x3B) / 16384.0
        g_y = self.lire_mot(0x3D) / 16384.0
        g_z = self.lire_mot(0x3F) / 16384.0
        return g_x, g_y, g_z


class AffichageG:
    def __init__(self, adresse=0x3C, limite_g=1.0):
        interface = i2c(port=1, address=adresse)
        self.ecran = ssd1306(interface)
        self.limite_g = limite_g
        self.centre_x = self.ecran.height // 2 + 2
        self.centre_y = self.ecran.height // 2
        self.rayon = self.centre_y - 4

    def mettre_a_jour(self, g_longitudinal, g_lateral, g_vertical):
        position_x = self.centre_x + int((g_lateral / self.limite_g) * self.rayon)
        position_y = self.centre_y - int((g_longitudinal / self.limite_g) * self.rayon)
        position_x = max(self.centre_x - self.rayon, min(self.centre_x + self.rayon, position_x))
        position_y = max(self.centre_y - self.rayon, min(self.centre_y + self.rayon, position_y))

        with canvas(self.ecran) as dessin:
            dessin.ellipse(
                (
                    self.centre_x - self.rayon,
                    self.centre_y - self.rayon,
                    self.centre_x + self.rayon,
                    self.centre_y + self.rayon,
                ),
                outline="white",
            )
            dessin.line(
                (self.centre_x, self.centre_y - self.rayon, self.centre_x, self.centre_y + self.rayon),
                fill="white",
            )
            dessin.line(
                (self.centre_x - self.rayon, self.centre_y, self.centre_x + self.rayon, self.centre_y),
                fill="white",
            )
            dessin.ellipse(
                (position_x - 3, position_y - 3, position_x + 3, position_y + 3),
                fill="white",
            )

            colonne_texte = self.centre_x + self.rayon + 8
            dessin.text((colonne_texte, 6), f"L {g_longitudinal:+.2f}", fill="white")
            dessin.text((colonne_texte, 26), f"T {g_lateral:+.2f}", fill="white")
            dessin.text((colonne_texte, 46), f"V {g_vertical:+.2f}", fill="white")


class EnregistreurExcel:
    def __init__(self, taille_lot=50):
        horodatage_session = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        self.nom_fichier = f"mesures_{horodatage_session}.xlsx"
        self.classeur = openpyxl.Workbook()
        self.feuille = self.classeur.active
        self.feuille.title = "Mesures"
        self.feuille.append(["horodatage", "g_longitudinal", "g_lateral", "g_vertical"])
        self.classeur.save(self.nom_fichier)
        self.tampon = []
        self.taille_lot = taille_lot

    def ajouter_mesure(self, g_longitudinal, g_lateral, g_vertical):
        horodatage = datetime.now().strftime("%Y-%m-%d %H:%M:%S.%f")
        self.tampon.append((horodatage, g_longitudinal, g_lateral, g_vertical))
        if len(self.tampon) >= self.taille_lot:
            self.vider_tampon()

    def vider_tampon(self):
        if not self.tampon:
            return
        for ligne in self.tampon:
            self.feuille.append(list(ligne))
        self.classeur.save(self.nom_fichier)
        self.tampon.clear()

    def fermer(self):
        self.vider_tampon()


en_cours = True


def arreter_proprement(signum, frame):
    global en_cours
    en_cours = False


def main():
    capteur = CapteurMPU6050()
    affichage = AffichageG()
    enregistreur = EnregistreurExcel()

    signal.signal(signal.SIGINT, arreter_proprement)
    signal.signal(signal.SIGTERM, arreter_proprement)

    frequence_affichage = 20
    periode = 1.0 / frequence_affichage

    while en_cours:
        g_longitudinal, g_lateral, g_vertical = capteur.lire_acceleration()
        affichage.mettre_a_jour(g_longitudinal, g_lateral, g_vertical)
        enregistreur.ajouter_mesure(g_longitudinal, g_lateral, g_vertical)
        time.sleep(periode)

    enregistreur.fermer()
    sys.exit(0)


if __name__ == "__main__":
    main()
