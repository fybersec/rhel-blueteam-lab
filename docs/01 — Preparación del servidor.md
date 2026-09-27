## Especificaciones de la VM

- **Hipervisor:** [VMWare](https://www.vmware.com/)
- **vCPUs asignadas:** [4](https://techdocs.broadcom.com/es/es/vmware-cis/vsphere/vsphere/9-0/vsphere-virtual-machine-administration/configuring-virtual-machine-hardwarevsphere-vm-admin/virtual-cpu-configuration-and-limitationsvsphere-vm-admin.html)
- **RAM asignada:** [8 GB](https://techdocs.broadcom.com/es/es/vmware-cis/desktop-hypervisors/workstation-pro/25H2/using-vmware-workstation-player-for-linux-17-0/configuring-and-managing-virtual-machines-linux/change-the-memory-allocation-for-a-virtual-machine-linux.html)
- **Disco:** [80 GB](https://cloudian.com/guides/vmware-storage/vmware-storage/)
- **Tipo de red virtual:** [Bridged](https://www.redeszone.net/tutoriales/redes-cable/configuracion-red-maquina-virtual-vmware/)
    - Justificación: A pesar de ser bridged, hay un aislamiento del resto de la red doméstica, necesidad de que la VM de Pentesting (Kali) y el servidor RHEL se vean entre sí sin salir a internet durante los ataques.
- **Snapshot inicial:** Recomendado para poder volver atrás si algo se rompe.

## Fuente de la imagen RHEL

- **Versión exacta:** RHEL 8.10
- **Origen de la ISO:** [rhel-8.10-x86_64-dvd](https://developers.redhat.com/products/rhel/download#downloadsbyrelease)

### Usuario y acceso inicial

| Usuario   | SSH-Login | Grupo | Autenticación  | Propósito              |
| :-------- | :-------: | ----- | -------------- | ---------------------- |
| root      |     -     | -     | -              | Cuenta del sistema     |
| soc-admin |    SI     | wheel | password/clave | Administración del lab |
## Red y hostname

- **Modo de IP:** [Estática](https://www.redhat.com/en/blog/static-dynamic-ip-1)
    - Justificación: En un lab de detección, una IP estática facilita correlacionar logs y reglas de Suricata/firewalld sin que cambie la dirección entre reinicios.

## Verificación post-instalación

Lista de lo que comprobaste que funcionara antes de pasar a `03-hardening.md`:

- [ ] Conectividad de red entre el servidor RHEL y la máquina de administración remota (Windows).
- [ ] Conectividad entre el servidor RHEL y la VM de Pentesting (Kali), dentro de la red aislada.
- [ ] Acceso SSH inicial funcional (con las credenciales por defecto, antes de endurecer nada).
- [ ] `dnf update` / verificación de repositorios habilitados. 
- [ ] Hora del sistema sincronizada (relevante para correlación de logs más adelante).

## Capturas

<p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/01-preparacion/1.png" width="600"> </p> <p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/01-preparacion/2.png" width="600"> </p> <p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/01-preparacion/3.png" width="600"> </p> <p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/01-preparacion/4.png" width="600"> </p> <p align="center"> <img src="https://github.com/fybersec/rhel-blueteam-lab/blob/main/screenshots-videos/assets/01-preparacion/5.png" width="600"> </p>
