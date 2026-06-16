==========================================================================================
RobotComQt.pro
==========================================================================================
QT += core gui widgets serialport

CONFIG += c++17
TEMPLATE = app
TARGET = RobotComQt

SOURCES += \
    main.cpp \
    mainwindow.cpp

HEADERS += \
    mainwindow.h \
    robotprotocol.h

# Р”Р»СЏ Qt 5.12 + MinGW. Р•СЃР»Рё РєРѕРЅРєСЂРµС‚РЅС‹Р№ РєРѕРјРїР»РµРєС‚ Qt РЅРµ РїРѕРЅРёРјР°РµС‚ CONFIG += c++17,
# Р·Р°РјРµРЅРёС‚Рµ СЃС‚СЂРѕРєСѓ РІС‹С€Рµ РЅР°: QMAKE_CXXFLAGS += -std=gnu++17


==========================================================================================
main.cpp
==========================================================================================
#include "mainwindow.h"

#include <QtWidgets/QApplication>

int main(int argc, char *argv[])
{
    QApplication application(argc, argv);

    MainWindow window;
    window.show();

    return application.exec();
}


==========================================================================================
mainwindow.h
==========================================================================================
#pragma once

#include "robotprotocol.h"

#include <QtSerialPort/QSerialPort>
#include <QtWidgets/QMainWindow>

class QCheckBox;
class QComboBox;
class QLabel;
class QLineEdit;
class QPushButton;
class QSpinBox;
class QTableWidget;
class QTextEdit;

class MainWindow final : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = nullptr);
    ~MainWindow() override = default;

private slots:
    void refreshPorts();
    void toggleConnection();
    void onSerialReadyRead();
    void onSerialError(QSerialPort::SerialPortError error);

    void addHeaderFromEditor();
    void removeSelectedHeader();
    void clearHeaders();
    void sendCurrentFrame();

    void fillGetParamExample();
    void fillSetParamExample();
    void clearLog();

private:
    void createUi();
    void fillProtocolCombos();
    void updateConnectionUi();
    void updateHeadersTable();

    RobotProtocol::Device selectedDevice(const QComboBox *combo) const;
    RobotProtocol::Request selectedRequest(const QComboBox *combo) const;

    void appendLog(const QString &text);
    void logParsedFrame(const RobotProtocol::ParseResult &result);
    QString currentTimestamp() const;

private:
    QSerialPort m_serial;
    RobotProtocol::StreamParser m_streamParser;
    QVector<RobotProtocol::Header> m_outgoingHeaders;

    QComboBox *m_portCombo = nullptr;
    QComboBox *m_baudCombo = nullptr;
    QPushButton *m_refreshPortsButton = nullptr;
    QPushButton *m_connectButton = nullptr;
    QLabel *m_connectionStateLabel = nullptr;

    QComboBox *m_destinationCombo = nullptr;
    QComboBox *m_sourceCombo = nullptr;
    QCheckBox *m_dummyByteCheckBox = nullptr;

    QComboBox *m_requestCombo = nullptr;
    QComboBox *m_headerDeviceCombo = nullptr;
    QSpinBox *m_headerStatusSpin = nullptr;
    QLineEdit *m_headerDataEdit = nullptr;
    QPushButton *m_addHeaderButton = nullptr;
    QPushButton *m_removeHeaderButton = nullptr;
    QPushButton *m_clearHeadersButton = nullptr;
    QTableWidget *m_headersTable = nullptr;

    QSpinBox *m_paramIdSpin = nullptr;
    QLineEdit *m_paramValuesEdit = nullptr;
    QPushButton *m_getParamExampleButton = nullptr;
    QPushButton *m_setParamExampleButton = nullptr;

    QPushButton *m_sendButton = nullptr;
    QTextEdit *m_logEdit = nullptr;
    QPushButton *m_clearLogButton = nullptr;
};


==========================================================================================
mainwindow.cpp
==========================================================================================
#include "mainwindow.h"

#include <QtCore/QDateTime>
#include <QtCore/QRegExp>
#include <QtCore/QStringList>
#include <QtSerialPort/QSerialPortInfo>
#include <QtWidgets/QAbstractItemView>
#include <QtWidgets/QCheckBox>
#include <QtWidgets/QComboBox>
#include <QtWidgets/QFormLayout>
#include <QtWidgets/QGroupBox>
#include <QtWidgets/QHeaderView>
#include <QtWidgets/QHBoxLayout>
#include <QtWidgets/QLabel>
#include <QtWidgets/QLineEdit>
#include <QtWidgets/QMessageBox>
#include <QtWidgets/QPushButton>
#include <QtWidgets/QSpinBox>
#include <QtWidgets/QSplitter>
#include <QtWidgets/QTableWidget>
#include <QtWidgets/QTextEdit>
#include <QtWidgets/QVBoxLayout>
#include <QtWidgets/QWidget>

#include <limits>

using namespace RobotProtocol;

namespace {

QVector<qint32> parseInt32Values(const QString &text, bool &ok, QString &error)
{
    QVector<qint32> values;
    ok = true;
    error.clear();

    const QString trimmed = text.trimmed();
    if (trimmed.isEmpty())
        return values;

    const QStringList parts = trimmed.split(
        QRegExp(QStringLiteral("[,;\\s]+")),
        Qt::SkipEmptyParts);

    for (const QString &part : parts) {
        bool valueOk = false;
        const qlonglong parsed = part.toLongLong(&valueOk, 0);

        if (!valueOk || parsed < std::numeric_limits<qint32>::min()
            || parsed > std::numeric_limits<qint32>::max()) {
            ok = false;
            error = QStringLiteral("'%1' РЅРµ СЏРІР»СЏРµС‚СЃСЏ РґРѕРїСѓСЃС‚РёРјС‹Рј int32_t.").arg(part);
            return {};
        }

        values.append(static_cast<qint32>(parsed));
    }

    return values;
}

} // namespace

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
{
    createUi();
    fillProtocolCombos();
    refreshPorts();
    updateConnectionUi();
    updateHeadersTable();

    connect(&m_serial, &QSerialPort::readyRead,
            this, &MainWindow::onSerialReadyRead);
    connect(&m_serial, &QSerialPort::errorOccurred,
            this, &MainWindow::onSerialError);

    setWindowTitle(QStringLiteral("Robot COM Protocol Demo"));
    resize(1120, 760);
}

void MainWindow::createUi()
{
    auto *central = new QWidget(this);
    auto *rootLayout = new QVBoxLayout(central);

    // -------------------------------------------------------------------------
    // COM-РїРѕСЂС‚
    // -------------------------------------------------------------------------
    auto *serialGroup = new QGroupBox(QStringLiteral("COM-РїРѕСЂС‚"), central);
    auto *serialLayout = new QHBoxLayout(serialGroup);

    m_portCombo = new QComboBox(serialGroup);
    m_portCombo->setMinimumWidth(180);

    m_refreshPortsButton = new QPushButton(QStringLiteral("РћР±РЅРѕРІРёС‚СЊ"), serialGroup);

    m_baudCombo = new QComboBox(serialGroup);
    const QList<int> baudRates = {
        9600, 19200, 38400, 57600, 115200, 230400, 460800, 921600
    };
    for (const int baudRate : baudRates)
        m_baudCombo->addItem(QString::number(baudRate), baudRate);
    m_baudCombo->setCurrentText(QStringLiteral("115200"));

    m_connectButton = new QPushButton(QStringLiteral("РџРѕРґРєР»СЋС‡РёС‚СЊСЃСЏ"), serialGroup);
    m_connectionStateLabel = new QLabel(QStringLiteral("РћС‚РєР»СЋС‡РµРЅРѕ"), serialGroup);

    serialLayout->addWidget(new QLabel(QStringLiteral("РџРѕСЂС‚:"), serialGroup));
    serialLayout->addWidget(m_portCombo);
    serialLayout->addWidget(m_refreshPortsButton);
    serialLayout->addSpacing(12);
    serialLayout->addWidget(new QLabel(QStringLiteral("Baud:"), serialGroup));
    serialLayout->addWidget(m_baudCombo);
    serialLayout->addWidget(m_connectButton);
    serialLayout->addWidget(m_connectionStateLabel);
    serialLayout->addStretch();

    rootLayout->addWidget(serialGroup);

    // -------------------------------------------------------------------------
    // РћСЃРЅРѕРІРЅР°СЏ РѕР±Р»Р°СЃС‚СЊ: СЂРµРґР°РєС‚РѕСЂ РєР°РґСЂР° + Р»РѕРі
    // -------------------------------------------------------------------------
    auto *splitter = new QSplitter(Qt::Vertical, central);

    auto *frameWidget = new QWidget(splitter);
    auto *frameLayout = new QVBoxLayout(frameWidget);
    frameLayout->setContentsMargins(0, 0, 0, 0);

    auto *addressGroup = new QGroupBox(QStringLiteral("РђРґСЂРµСЃР° РєР°РґСЂР°"), frameWidget);
    auto *addressLayout = new QHBoxLayout(addressGroup);

    m_destinationCombo = new QComboBox(addressGroup);
    m_sourceCombo = new QComboBox(addressGroup);
    m_dummyByteCheckBox = new QCheckBox(
        QStringLiteral("Р”РѕР±Р°РІР»СЏС‚СЊ С„РёРєС‚РёРІРЅС‹Р№ 0xFF РїРµСЂРµРґ СЃС‚Р°СЂС‚РѕРІС‹Рј C0"),
        addressGroup);
    m_dummyByteCheckBox->setChecked(true);

    addressLayout->addWidget(new QLabel(QStringLiteral("РџРѕР»СѓС‡Р°С‚РµР»СЊ DST:"), addressGroup));
    addressLayout->addWidget(m_destinationCombo);
    addressLayout->addSpacing(12);
    addressLayout->addWidget(new QLabel(QStringLiteral("РћС‚РїСЂР°РІРёС‚РµР»СЊ SRC:"), addressGroup));
    addressLayout->addWidget(m_sourceCombo);
    addressLayout->addSpacing(12);
    addressLayout->addWidget(m_dummyByteCheckBox);
    addressLayout->addStretch();

    frameLayout->addWidget(addressGroup);

    auto *headerGroup = new QGroupBox(QStringLiteral("РќРѕРІС‹Р№ header"), frameWidget);
    auto *headerLayout = new QFormLayout(headerGroup);

    m_requestCombo = new QComboBox(headerGroup);
    m_headerDeviceCombo = new QComboBox(headerGroup);
    m_headerStatusSpin = new QSpinBox(headerGroup);
    m_headerStatusSpin->setRange(0, 255);
    m_headerStatusSpin->setDisplayIntegerBase(16);
    m_headerStatusSpin->setPrefix(QStringLiteral("0x"));

    m_headerDataEdit = new QLineEdit(headerGroup);
    m_headerDataEdit->setPlaceholderText(
        QStringLiteral("HEX, РЅР°РїСЂРёРјРµСЂ: 34 12 01 00 00 00"));

    m_addHeaderButton = new QPushButton(QStringLiteral("Р”РѕР±Р°РІРёС‚СЊ header РІ РєР°РґСЂ"), headerGroup);

    headerLayout->addRow(QStringLiteral("request_id:"), m_requestCombo);
    headerLayout->addRow(QStringLiteral("device_id:"), m_headerDeviceCombo);
    headerLayout->addRow(QStringLiteral("status:"), m_headerStatusSpin);
    headerLayout->addRow(QStringLiteral("data[] (HEX):"), m_headerDataEdit);
    headerLayout->addRow(QString(), m_addHeaderButton);

    frameLayout->addWidget(headerGroup);

    auto *paramGroup = new QGroupBox(
        QStringLiteral("РџРѕРјРѕС‰РЅРёРє РґР»СЏ GetParam / SetParam"), frameWidget);
    auto *paramLayout = new QHBoxLayout(paramGroup);

    m_paramIdSpin = new QSpinBox(paramGroup);
    m_paramIdSpin->setRange(0, 65535);
    m_paramIdSpin->setDisplayIntegerBase(16);
    m_paramIdSpin->setPrefix(QStringLiteral("0x"));

    m_paramValuesEdit = new QLineEdit(paramGroup);
    m_paramValuesEdit->setPlaceholderText(
        QStringLiteral("int32 values С‡РµСЂРµР· РїСЂРѕР±РµР»/Р·Р°РїСЏС‚СѓСЋ, РЅР°РїСЂРёРјРµСЂ: 100 -25 0x7FFFFFFF"));

    m_getParamExampleButton = new QPushButton(
        QStringLiteral("РџРѕРґСЃС‚Р°РІРёС‚СЊ GetParam request"), paramGroup);
    m_setParamExampleButton = new QPushButton(
        QStringLiteral("РџРѕРґСЃС‚Р°РІРёС‚СЊ SetParam request"), paramGroup);

    paramLayout->addWidget(new QLabel(QStringLiteral("ID:"), paramGroup));
    paramLayout->addWidget(m_paramIdSpin);
    paramLayout->addWidget(new QLabel(QStringLiteral("Values:"), paramGroup));
    paramLayout->addWidget(m_paramValuesEdit, 1);
    paramLayout->addWidget(m_getParamExampleButton);
    paramLayout->addWidget(m_setParamExampleButton);

    frameLayout->addWidget(paramGroup);

    m_headersTable = new QTableWidget(frameWidget);
    m_headersTable->setColumnCount(5);
    m_headersTable->setHorizontalHeaderLabels({
        QStringLiteral("request_id"),
        QStringLiteral("device_id"),
        QStringLiteral("status"),
        QStringLiteral("data_size"),
        QStringLiteral("data HEX")
    });
    m_headersTable->setSelectionBehavior(QAbstractItemView::SelectRows);
    m_headersTable->setSelectionMode(QAbstractItemView::SingleSelection);
    m_headersTable->setEditTriggers(QAbstractItemView::NoEditTriggers);
    m_headersTable->horizontalHeader()->setSectionResizeMode(0, QHeaderView::ResizeToContents);
    m_headersTable->horizontalHeader()->setSectionResizeMode(1, QHeaderView::ResizeToContents);
    m_headersTable->horizontalHeader()->setSectionResizeMode(2, QHeaderView::ResizeToContents);
    m_headersTable->horizontalHeader()->setSectionResizeMode(3, QHeaderView::ResizeToContents);
    m_headersTable->horizontalHeader()->setSectionResizeMode(4, QHeaderView::Stretch);

    frameLayout->addWidget(m_headersTable, 1);

    auto *frameButtonsLayout = new QHBoxLayout();
    m_removeHeaderButton = new QPushButton(QStringLiteral("РЈРґР°Р»РёС‚СЊ РІС‹Р±СЂР°РЅРЅС‹Р№"), frameWidget);
    m_clearHeadersButton = new QPushButton(QStringLiteral("РћС‡РёСЃС‚РёС‚СЊ headers"), frameWidget);
    m_sendButton = new QPushButton(QStringLiteral("РћС‚РїСЂР°РІРёС‚СЊ РєР°РґСЂ"), frameWidget);

    frameButtonsLayout->addWidget(m_removeHeaderButton);
    frameButtonsLayout->addWidget(m_clearHeadersButton);
    frameButtonsLayout->addStretch();
    frameButtonsLayout->addWidget(m_sendButton);

    frameLayout->addLayout(frameButtonsLayout);

    auto *logWidget = new QWidget(splitter);
    auto *logLayout = new QVBoxLayout(logWidget);
    logLayout->setContentsMargins(0, 0, 0, 0);

    auto *logTopLayout = new QHBoxLayout();
    logTopLayout->addWidget(new QLabel(QStringLiteral("РћР±РјРµРЅ Рё СЂР°Р·Р±РѕСЂ РєР°РґСЂРѕРІ"), logWidget));
    logTopLayout->addStretch();
    m_clearLogButton = new QPushButton(QStringLiteral("РћС‡РёСЃС‚РёС‚СЊ Р»РѕРі"), logWidget);
    logTopLayout->addWidget(m_clearLogButton);

    m_logEdit = new QTextEdit(logWidget);
    m_logEdit->setReadOnly(true);
    m_logEdit->setLineWrapMode(QTextEdit::NoWrap);
    m_logEdit->setFontFamily(QStringLiteral("Consolas"));

    logLayout->addLayout(logTopLayout);
    logLayout->addWidget(m_logEdit, 1);

    splitter->addWidget(frameWidget);
    splitter->addWidget(logWidget);
    splitter->setStretchFactor(0, 3);
    splitter->setStretchFactor(1, 2);

    rootLayout->addWidget(splitter, 1);
    setCentralWidget(central);

    connect(m_refreshPortsButton, &QPushButton::clicked,
            this, &MainWindow::refreshPorts);
    connect(m_connectButton, &QPushButton::clicked,
            this, &MainWindow::toggleConnection);
    connect(m_addHeaderButton, &QPushButton::clicked,
            this, &MainWindow::addHeaderFromEditor);
    connect(m_removeHeaderButton, &QPushButton::clicked,
            this, &MainWindow::removeSelectedHeader);
    connect(m_clearHeadersButton, &QPushButton::clicked,
            this, &MainWindow::clearHeaders);
    connect(m_sendButton, &QPushButton::clicked,
            this, &MainWindow::sendCurrentFrame);
    connect(m_getParamExampleButton, &QPushButton::clicked,
            this, &MainWindow::fillGetParamExample);
    connect(m_setParamExampleButton, &QPushButton::clicked,
            this, &MainWindow::fillSetParamExample);
    connect(m_clearLogButton, &QPushButton::clicked,
            this, &MainWindow::clearLog);
}

void MainWindow::fillProtocolCombos()
{
    for (const Device device : allDevices()) {
        const int raw = static_cast<quint8>(device);
        const QString text = deviceDisplayName(device);
        m_destinationCombo->addItem(text, raw);
        m_sourceCombo->addItem(text, raw);
        m_headerDeviceCombo->addItem(text, raw);
    }

    for (const Request request : allRequests()) {
        m_requestCombo->addItem(
            requestDisplayName(request),
            static_cast<quint8>(request));
    }

    // РЈРґРѕР±РЅС‹Рµ СЃС‚Р°СЂС‚РѕРІС‹Рµ Р·РЅР°С‡РµРЅРёСЏ РґР»СЏ РїСЂРёРјРµСЂР°:
    // РџРљ/РІРЅРµС€РЅРµРµ СЃРѕРµРґРёРЅРµРЅРёРµ -> РѕР±С‰РёР№ РєРѕРЅС‚СЂРѕР»Р»РµСЂ.
    const int destinationIndex = m_destinationCombo->findData(
        static_cast<quint8>(Device::General));
    if (destinationIndex >= 0)
        m_destinationCombo->setCurrentIndex(destinationIndex);

    const int sourceIndex = m_sourceCombo->findData(
        static_cast<quint8>(Device::ExternalConn));
    if (sourceIndex >= 0)
        m_sourceCombo->setCurrentIndex(sourceIndex);

    const int headerDeviceIndex = m_headerDeviceCombo->findData(
        static_cast<quint8>(Device::General));
    if (headerDeviceIndex >= 0)
        m_headerDeviceCombo->setCurrentIndex(headerDeviceIndex);

    const int getStatusIndex = m_requestCombo->findData(
        static_cast<quint8>(Request::GetStatus));
    if (getStatusIndex >= 0)
        m_requestCombo->setCurrentIndex(getStatusIndex);
}

void MainWindow::refreshPorts()
{
    const QString previousPort = m_portCombo->currentData().toString();
    m_portCombo->clear();

    const QList<QSerialPortInfo> ports = QSerialPortInfo::availablePorts();
    for (const QSerialPortInfo &info : ports) {
        QString title = info.portName();
        if (!info.description().isEmpty())
            title += QStringLiteral(" вЂ” ") + info.description();

        m_portCombo->addItem(title, info.portName());
    }

    const int previousIndex = m_portCombo->findData(previousPort);
    if (previousIndex >= 0)
        m_portCombo->setCurrentIndex(previousIndex);

    if (ports.isEmpty())
        appendLog(QStringLiteral("COM-РїРѕСЂС‚С‹ РЅРµ РЅР°Р№РґРµРЅС‹."));
}

void MainWindow::toggleConnection()
{
    if (m_serial.isOpen()) {
        m_serial.close();
        m_streamParser.reset();
        appendLog(QStringLiteral("COM-РїРѕСЂС‚ Р·Р°РєСЂС‹С‚."));
        updateConnectionUi();
        return;
    }

    if (m_portCombo->currentIndex() < 0) {
        QMessageBox::warning(this, QStringLiteral("COM-РїРѕСЂС‚"),
                             QStringLiteral("РЎРЅР°С‡Р°Р»Р° РІС‹Р±РµСЂРёС‚Рµ COM-РїРѕСЂС‚."));
        return;
    }

    const QString portName = m_portCombo->currentData().toString();
    const int baudRate = m_baudCombo->currentData().toInt();

    m_serial.setPortName(portName);
    m_serial.setBaudRate(baudRate);
    m_serial.setDataBits(QSerialPort::Data8);
    m_serial.setParity(QSerialPort::NoParity);
    m_serial.setStopBits(QSerialPort::OneStop);
    m_serial.setFlowControl(QSerialPort::NoFlowControl);

    m_streamParser.reset();

    if (!m_serial.open(QIODevice::ReadWrite)) {
        QMessageBox::critical(
            this,
            QStringLiteral("РћС€РёР±РєР° COM-РїРѕСЂС‚Р°"),
            QStringLiteral("РќРµ СѓРґР°Р»РѕСЃСЊ РѕС‚РєСЂС‹С‚СЊ %1:\n%2")
                .arg(portName, m_serial.errorString()));
        updateConnectionUi();
        return;
    }

    appendLog(QStringLiteral("РћС‚РєСЂС‹С‚ %1, %2 baud, 8N1, flow control: none.")
                  .arg(portName)
                  .arg(baudRate));
    updateConnectionUi();
}

void MainWindow::onSerialReadyRead()
{
    const QByteArray chunk = m_serial.readAll();
    if (chunk.isEmpty())
        return;

    appendLog(QStringLiteral("RX chunk (%1 bytes): %2")
                  .arg(chunk.size())
                  .arg(bytesToHex(chunk)));

    const QVector<StreamEvent> events = m_streamParser.feed(chunk);
    for (const StreamEvent &event : events)
        logParsedFrame(event.parsed);
}

void MainWindow::onSerialError(QSerialPort::SerialPortError error)
{
    if (error == QSerialPort::NoError)
        return;

    appendLog(QStringLiteral("РћС€РёР±РєР° COM-РїРѕСЂС‚Р°: %1").arg(m_serial.errorString()));

    // ResourceError РѕР±С‹С‡РЅРѕ РѕР·РЅР°С‡Р°РµС‚ РѕС‚РєР»СЋС‡РµРЅРёРµ USB-UART РёР»Рё РёСЃС‡РµР·РЅРѕРІРµРЅРёРµ РїРѕСЂС‚Р°.
    if (error == QSerialPort::ResourceError && m_serial.isOpen()) {
        m_serial.close();
        m_streamParser.reset();
        updateConnectionUi();
    }
}

void MainWindow::addHeaderFromEditor()
{
    QByteArray data;
    QString parseError;

    if (!hexToBytes(m_headerDataEdit->text(), data, parseError)) {
        QMessageBox::warning(this, QStringLiteral("РќРµРІРµСЂРЅС‹Р№ HEX"), parseError);
        return;
    }

    if (data.size() > 255) {
        QMessageBox::warning(
            this,
            QStringLiteral("РЎР»РёС€РєРѕРј Р±РѕР»СЊС€РѕР№ payload"),
            QStringLiteral("data_size вЂ” uint8_t, РїРѕСЌС‚РѕРјСѓ РјР°РєСЃРёРјСѓРј 255 Р±Р°Р№С‚."));
        return;
    }

    Header header;
    header.request = selectedRequest(m_requestCombo);
    header.devi